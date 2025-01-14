




// server/models/Event.js
const mongoose = require('mongoose');

const eventSchema = new mongoose.Schema({
  title: { type: String, required: true },
  description: String,
  date: { type: Date, required: true },
  venue: { type: String, required: true },
  totalTickets: { type: Number, required: true },
  availableTickets: { type: Number, required: true },
  price: { type: Number, required: true },
  category: String,
  image: String,
  createdAt: { type: Date, default: Date.now },
});

const Event = mongoose.model('Event', eventSchema);

// server/models/Ticket.js
const ticketSchema = new mongoose.Schema({
  eventId: { type: mongoose.Schema.Types.ObjectId, ref: 'Event', required: true },
  userId: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
  purchaseDate: { type: Date, default: Date.now },
  status: { 
    type: String, 
    enum: ['active', 'used', 'cancelled'],
    default: 'active'
  },
  ticketNumber: String,
  price: Number,
});

const Ticket = mongoose.model('Ticket', ticketSchema);

// server/routes/events.js
const express = require('express');
const router = express.Router();
const Event = require('../models/Event');
const auth = require('../middleware/auth');

router.get('/', async (req, res) => {
  try {
    const { category, search, date } = req.query;
    let query = {};
    
    if (category) query.category = category;
    if (search) {
      query.$or = [
        { title: { $regex: search, $options: 'i' } },
        { description: { $regex: search, $options: 'i' } }
      ];
    }
    if (date) query.date = { $gte: new Date(date) };

    const events = await Event.find(query).sort({ date: 1 });
    res.json(events);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

router.post('/', auth, async (req, res) => {
  try {
    const event = new Event({
      ...req.body,
      availableTickets: req.body.totalTickets
    });
    const newEvent = await event.save();
    res.status(201).json(newEvent);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

// server/routes/tickets.js
router.post('/purchase', auth, async (req, res) => {
  const session = await mongoose.startSession();
  session.startTransaction();
  
  try {
    const { eventId, quantity } = req.body;
    const event = await Event.findById(eventId);
    
    if (!event) throw new Error('Event not found');
    if (event.availableTickets < quantity) {
      throw new Error('Not enough tickets available');
    }

    const tickets = [];
    for (let i = 0; i < quantity; i++) {
      const ticket = new Ticket({
        eventId,
        userId: req.user._id,
        price: event.price,
        ticketNumber: generateTicketNumber()
      });
      tickets.push(ticket);
    }

    await Ticket.insertMany(tickets, { session });
    event.availableTickets -= quantity;
    await event.save({ session });
    
    await session.commitTransaction();
    res.status(201).json(tickets);
  } catch (error) {
    await session.abortTransaction();
    res.status(400).json({ message: error.message });
  } finally {
    session.endSession();
  }
});

// client/src/components/EventList.jsx
import React, { useState, useEffect } from 'react';
import axios from 'axios';

const EventList = () => {
  const [events, setEvents] = useState([]);
  const [filters, setFilters] = useState({
    category: '',
    search: '',
    date: ''
  });

  useEffect(() => {
    const fetchEvents = async () => {
      try {
        const params = new URLSearchParams(filters);
        const response = await axios.get(`/api/events?${params}`);
        setEvents(response.data);
      } catch (error) {
        console.error('Error fetching events:', error);
      }
    };

    fetchEvents();
  }, [filters]);

  return (
    <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6 p-6">
      <div className="mb-6 col-span-full">
        <input
          type="text"
          placeholder="Search events..."
          className="p-2 border rounded"
          onChange={(e) => setFilters({ ...filters, search: e.target.value })}
        />
        <select
          className="ml-4 p-2 border rounded"
          onChange={(e) => setFilters({ ...filters, category: e.target.value })}
        >
          <option value="">All Categories</option>
          <option value="concert">Concerts</option>
          <option value="sports">Sports</option>
          <option value="theater">Theater</option>
        </select>
      </div>
      
      {events.map(event => (
        <div key={event._id} className="border rounded-lg overflow-hidden shadow-lg">
          {event.image && (
            <img src={event.image} alt={event.title} className="w-full h-48 object-cover" />
          )}
          <div className="p-4">
            <h3 className="text-xl font-bold mb-2">{event.title}</h3>
            <p className="text-gray-600 mb-2">{event.venue}</p>
            <p className="text-gray-600 mb-4">
              {new Date(event.date).toLocaleDateString()}
            </p>
            <div className="flex justify-between items-center">
              <span className="text-lg font-bold">${event.price}</span>
              <button 
                className="bg-blue-500 text-white px-4 py-2 rounded hover:bg-blue-600"
                disabled={event.availableTickets === 0}
              >
                {event.availableTickets > 0 ? 'Buy Tickets' : 'Sold Out'}
              </button>
            </div>
          </div>
        </div>
      ))}
    </div>
  );
};

// client/src/components/PurchaseModal.jsx
const PurchaseModal = ({ event, onClose, onPurchase }) => {
  const [quantity, setQuantity] = useState(1);
  const [isProcessing, setIsProcessing] = useState(false);

  const handlePurchase = async () => {
    try {
      setIsProcessing(true);
      await onPurchase(quantity);
      onClose();
    } catch (error) {
      console.error('Purchase failed:', error);
    } finally {
      setIsProcessing(false);
    }
  };

  return (
    <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center">
      <div className="bg-white p-6 rounded-lg max-w-md w-full">
        <h2 className="text-xl font-bold mb-4">{event.title}</h2>
        <div className="mb-4">
          <label className="block mb-2">Number of tickets:</label>
          <select 
            value={quantity} 
            onChange={(e) => setQuantity(Number(e.target.value))}
            className="w-full p-2 border rounded"
          >
            {[...Array(Math.min(event.availableTickets, 10))].map((_, i) => (
              <option key={i + 1} value={i + 1}>{i + 1}</option>
            ))}
          </select>
        </div>
        <div className="mb-4">
          <p className="text-lg">Total: ${(event.price * quantity).toFixed(2)}</p>
        </div>
        <div className="flex justify-end gap-4">
          <button
            onClick={onClose}
            className="px-4 py-2 border rounded hover:bg-gray-100"
          >
            Cancel
          </button>
          <button
            onClick={handlePurchase}
            disabled={isProcessing}
            className="px-4 py-2 bg-blue-500 text-white rounded hover:bg-blue-600"
          >
            {isProcessing ? 'Processing...' : 'Confirm Purchase'}
          </button>
        </div>
      </div>
    </div>
  );
};

```

Overview of the key components:

Backend:
1. Models:
   - Event: Stores event details, including title, venue, tickets, pricing
   - Ticket: Manages ticket purchases and status

2. Routes:
   - Events API for listing and creating events
   - Tickets API for purchasing tickets with transaction support

Frontend:
1. Event List:
   - Grid display of events
   - Search and category filters
   - Responsive design

2. Purchase Modal:
   - Ticket quantity selection
   - Price calculation
   - Purchase confirmation

Key Features:
- Event searching and filtering
- Ticket inventory management
- Secure ticket purchases with transaction support
- Responsive UI with Tailwind CSS

To complete the implementation, you would need to:

1. Set up MongoDB and Express server
2. Add user authentication
3. Implement payment processing
4. Add email confirmation system
5. Set up image upload for event photos
