<template>
  <div :class="['notifications-container', { 'dark-mode': isDarkMode }]">
    <!-- Back Button -->
    <router-link to="/dashboard" class="back-button">
      ← Back
    </router-link>  

    <!-- Title -->
    <h2 class="notifications-title">Notifications</h2>

    <!-- Clear Notifications Button -->
    <button class="clear-btn" @click="clearNotifications">Clear Notifications</button>

    <!-- Notifications List -->
    <div class="notifications-list">
      <!-- Check if there are no notifications -->
      <p v-if="notifications.length === 0" class="no-notifications">
        No notifications at the moment.
      </p>

      <!-- Render notifications if there are any -->
      <div v-for="(notification, index) in notifications" :key="index" class="notification-item">
        <p class="order-id">Order ID: {{ notification.orderId }}</p>
        <!-- Render message with highlighted details -->
        <p class="notification-message" v-html="formatHighlightedMessage(notification.message)"></p>
        
        <!-- Time Ago -->
        <p class="notification-time">{{ formatTimeAgo(notification.timestamp) }}</p>
        
        <hr />
      </div>
    </div>
  </div>
</template>


<script>
import { eventBus } from "@/utils/eventBus"; // Correct the path if needed

export default {
  data() {
    return {
      notifications: [],
      ws: null,
      wsConnected: false,
      isDarkMode: localStorage.getItem("darkMode") === "true",
    };
  },
  methods: {
    formatTimeAgo(timestamp) {
      const now = new Date();
      const diff = now - new Date(timestamp);
      const minutes = Math.floor(diff / 1000 / 60);
      const hours = Math.floor(diff / 1000 / 60 / 60);
      const days = Math.floor(diff / 1000 / 60 / 60 / 24);

      if (days > 0) return `${days} day${days > 1 ? "s" : ""} ago`;
      if (hours > 0) return `${hours} hour${hours > 1 ? "s" : ""} ago`;
      if (minutes > 0) return `${minutes} minute${minutes > 1 ? "s" : ""} ago`;
      return "Just now";
    },

    updateNotificationCount() {
      const unreadCount = this.notifications.filter(notification => !notification.read).length;
      eventBus.notificationsCount = unreadCount; 
    },

    fetchNotifications() {
      const userName = localStorage.getItem("userName"); 
      if (userName) {
        const userNotificationsKey = `user_notifications_${userName}`;
        const storedNotifications = JSON.parse(localStorage.getItem(userNotificationsKey)) || [];
        
        // Sort notifications by timestamp (newest first)
        storedNotifications.sort((a, b) => {
          const dateA = new Date(a.timestamp);
          const dateB = new Date(b.timestamp);
          return dateB - dateA;
        });
        
        // Update the notifications array with the sorted list
        this.notifications = storedNotifications;
        this.updateNotificationCount();
      } else {
        this.notifications = [];
      }
    },

    addNewNotification(notification) {
      const userName = localStorage.getItem("userName");
      const userNotificationsKey = `user_notifications_${userName}`;
      let notifications = JSON.parse(localStorage.getItem(userNotificationsKey)) || [];
      
      // Always ensure the notification has the current timestamp if not provided
      if (!notification.timestamp) {
        notification.timestamp = new Date().toISOString();
      }
      
      // Check if a notification with the same order ID already exists
      const existingIndex = notifications.findIndex(n => n.orderId === notification.orderId);
      
      if (existingIndex !== -1) {
        // Instead of replacing, add as a new notification if the message is different
        if (notifications[existingIndex].message !== notification.message) {
          notifications.unshift(notification);
        }
      } else {
        // Add the new notification
        notifications.unshift(notification);
      }
      
      // Sort notifications by timestamp (newest first)
      notifications.sort((a, b) => {
        const dateA = new Date(a.timestamp);
        const dateB = new Date(b.timestamp);
        return dateB - dateA;
      });
      
      // Save the updated notifications to localStorage
      localStorage.setItem(userNotificationsKey, JSON.stringify(notifications));

      // Update the notifications in the component
      this.notifications = notifications;
      
      // Update notification count
      this.updateNotificationCount();
      
      // Dispatch event to notify other components
      window.dispatchEvent(new Event("notificationUpdated"));
    },

    formatHighlightedMessage(message) {
      // Enhanced message formatting to handle multiple order details sections
      return message.replace(
        /(Order details:.*?Total: ₱\d+(\.\d{2})?)/g,
        `<br/><br/><span class="highlighted-order-details">$1</span>`
      );
    },

    clearNotifications() {
      const userName = localStorage.getItem("userName");
      if (userName) {
        const userNotificationsKey = `user_notifications_${userName}`;
        localStorage.removeItem(userNotificationsKey);
        this.notifications = [];
        localStorage.setItem("unread_notifications", 0);
        this.updateNotificationCount();
      }
    },

    initWebSocket() {
      const wsUrl = `ws://${window.location.hostname}:8000/ws/orders`;    
      this.ws = new WebSocket(wsUrl);
      
      this.ws.onopen = () => {
        console.log('WebSocket connected');
        this.wsConnected = true;
      };
      
      this.ws.onmessage = (event) => {
        try {
          const data = JSON.parse(event.data);
          console.log('WebSocket message received:', data);
          
          if (data.type === 'new_order') {
            // Handle new order notification
            const userName = localStorage.getItem("userName");
            if (data.order.customer_name === userName) {
              this.addNewNotification({
                orderId: data.order.id,
                message: `Your order has been received! and is now being prepared. We will notify you as soon as it is ready for pickup.  Order details: ${this.formatItems(data.order.items)}. Total: ₱${this.calculateTotal(data.order.items)}`,
                timestamp: data.order.created_at
              });
            }
          } else if (data.type === 'order_status_update') {
            // Handle order status update notification
            const userName = localStorage.getItem("userName");
            if (data.customer_name === userName) {
              this.addNewNotification({
                orderId: data.order_id,
                message: `Your order status has been updated to ${data.status}`,
                timestamp: new Date().toISOString()
              });
            }
          }
        } catch (error) {
          console.error('Error processing WebSocket message:', error);
        }
      };
      
      this.ws.onclose = () => {
        console.log('WebSocket disconnected');
        this.wsConnected = false;
        // Try to reconnect after 5 seconds
        setTimeout(() => {
          this.initWebSocket();
        }, 5000);
      };
      
      this.ws.onerror = (error) => {
        console.error('WebSocket error:', error);
        this.wsConnected = false;
      };
    },

    formatItems(items) {
      return items.map(item => `${item.name} x${item.quantity}`).join(", ");
    },

    calculateTotal(items) {
      return items.reduce((sum, item) => sum + (item.price * item.quantity), 0).toFixed(2);
    }
  },
  created() {
    this.fetchNotifications();
    this.initWebSocket();
    window.addEventListener("notificationUpdated", this.fetchNotifications);
  },
  beforeUnmount() {
    window.removeEventListener("notificationUpdated", this.fetchNotifications);
    if (this.ws) {
      this.ws.close();
    }
  }
};
</script>






<style scoped>
/* Highlighted Order Details */
.highlighted-order-details {
  display: inline-block;
  background-color: #f8d2e4;  /* Light pink background */
  color: #d12f7a;  /* Bold color for text */
  padding: 10px;
  border-radius: 5px;
  margin-top: 10px;
  font-weight: bold;
  border: 2px solid #d12f7a;  /* Adding a border to make it stand out */
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  width: 100%;
  text-align: center;
}

/* Optional: Adding an underline to make it more prominent */
.highlighted-order-details::before {
  content: "——— ";
  font-size: 24px;
  font-weight: bold;
  color: #d12f7a;
}

/* Dark Mode Styles */
.dark-mode .notifications-container {
  background-color: #333;
  color: white;  /* Light text for dark mode */
}
.dark-mode .notification-time {
  color: #ccc;
}



.dark-mode .notification-item {
  background-color: #444;
  color: white; /* Light text color for dark background */
}

.dark-mode .order-id,
.dark-mode .no-notifications,
.dark-mode .notification-message {
  color: white; /* Lighter text color in dark mode */
}

.dark-mode .back-button {
  color: white; /* White text for the back button */
}

.dark-mode .highlighted-order-details {
  background-color: #555;
  color: #ffcc00;
  border: 2px solid #ffcc00;
}

/* Default Light Mode Styles */
.notifications-container {
  padding: 20px;
  background-color: #fce6e6;
  height: 100vh;
  display: flex;
  flex-direction: column;
}

.back-button {
  font-size: 18px;
  color: #333;
  margin-bottom: 10px;
  text-decoration: none;
}

.notifications-title {
  font-size: 24px;
  font-weight: bold;
  color: #white;
  margin-bottom: 20px;
  text-align: center;
}

.clear-btn {
  background-color: rgb(66, 47, 56);
  color: white;
  padding: 8px 16px;
  border: none;
  font-size: 16px;
  border-radius: 5px;
  cursor: pointer;
  align-self: flex-end;
  margin-bottom: 20px;
}

.clear-btn:hover {
  background-color: #b82d67;
}

.notifications-list {
  flex-grow: 1;
  overflow-y: auto;
}

.notification-item {
  background-color: #e6e6e6;
  margin-bottom: 20px;
  padding: 15px;
  border-radius: 10px;
}

.order-id {
  font-weight: bold;
  color: #333;
  font-size: 20px;
}

.notification-message {
  color: #333;
  font-size: 19px;
}

.notification-time {
  font-size: 14px;
  color: #222; /* Lighter color for the time ago text */
  margin-top: 10px;
  font-style: italic;
  text-align: right; /* Align time to the right */
}

hr {
  margin: 10px 0;
  border-color: #ccc;
}

.no-notifications {
  font-size: 18px;
  color: rgb(68, 68, 68);
  font-weight: bold;
  text-align: center;
  margin-top: 20px;
  padding: 15px;
  border-radius: 5px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
}
</style>
