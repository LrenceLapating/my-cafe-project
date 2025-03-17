<template>
  <div class="dashboard">
    <!-- Sidebar Toggle Button (For Mobile) -->
    <button class="menu-button" @click="toggleSidebar">
      <div class="menu-icon-container">
        ≡
        <span v-if="unreadNotificationsCount > 0" class="menu-notification-badge">{{ unreadNotificationsCount }}</span>
      </div>
    </button>

    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.15.4/css/all.min.css">
    
    <div v-if="isSidebarOpen" class="overlay" @click="closeSidebar"></div>
    
    <!-- Sidebar -->
    <div :class="['sidebar', { 
      'open': isSidebarOpen, 
      'light-mode': !isDarkMode,
      'dark-mode': isDarkMode 
    }]" @click.stop>
      <button class="close-sidebar" @click="toggleSidebar">✕</button>
      
      <!-- User Profile Section -->
      <div class="user-profile-section">
        <div class="profile-image">
          <img :src="userProfileImage || require('@/assets/default.png')" alt="Profile Picture">
        </div>
        <div class="profile-info">
          <span class="user-name">{{ userName }}</span>
        </div>
      </div>

      <!-- Dark Mode Toggle beside profile -->
      <button class="theme-toggle" @click="toggleDarkMode">
        <i :class="['fas', isDarkMode ? 'fa-sun' : 'fa-moon', { 'light-icon': !isDarkMode, 'dark-icon': isDarkMode }]"></i>
      </button>

      <!-- Profile Section -->
      <button class="profile-section" @click="handleProfile">
        <i class="fas fa-user"></i>
        <span>Profile</span>
      </button>

      <hr class="utility-divider">

      <!-- Utility Buttons (Removed Dark Mode from here) -->
      <div class="utility-section">
        <router-link to="/user-notifications" class="utility-button notification-link">
          <div class="notification-icon">
            <i class="fas fa-bell"></i>
            <span v-if="unreadNotificationsCount > 0" class="notification-badge">{{ unreadNotificationsCount }}</span>
          </div>
          <span>Notifications</span>
        </router-link>

        <button class="utility-button" @click="handleOrderHistory">
          <i class="fas fa-history"></i>
          <span>Order History</span>
        </button>
      </div>

      <!-- Categories -->
      <div class="menu-section">
        <!-- All Menu Items button -->
        <button 
          class="menu-item all-items-button" 
          @click="filterCategory('All')"
        >
          <i class="fas fa-utensils"></i>
          <span>All Menu Items</span>
        </button>

        <h3>Drinks</h3>
        <div class="menu-items">
          <button 
            v-for="category in drinkCategories" 
            :key="category"
            class="menu-item" 
            @click="filterCategory(category)"
          >
            <i :class="getCategoryIcon(category)"></i>
            <span>{{ category }}</span>
          </button>
        </div>

        <h3>Food</h3>
        <div class="menu-items">
          <button 
            v-for="category in foodCategories" 
            :key="category"
            class="menu-item" 
            @click="filterCategory(category)"
          >
            <i :class="getCategoryIcon(category)"></i>
            <span>{{ category }}</span>
          </button>
        </div>
      </div>

      <!-- Logout Button -->
      <div class="logout-container">
        <button class="utility-button logout" @click="handleLogout">
          <i class="fas fa-sign-out-alt"></i>
          <span>Logout</span>
        </button>
      </div>
    </div>

    <!-- Main content area -->
    <div :class="['content', { 'close': isSidebarOpen }]" @click="closeSidebar">
      <!-- Top Bar -->
      <div class="top-bar">
        <div class="logo-time-container">
          <div class="logo-container">
            <img src="@/assets/cafe-logo1.png" alt="University Logo" class="logo logo-light" />
          </div>
          <div class="live-time">
            <p>{{ currentTime }}</p>
          </div>
        </div>

        <div class="search-cart-container">
          <div class="search-container">
            <input
              type="text"
              v-model="searchQuery"
              placeholder="Search our Drinks and Food"
              @input="filterItems"
            />
          </div>
          <div class="cart-icon-container" @click="goToCart">
            <i class="fas fa-shopping-cart"></i>
            <span v-if="cartItemCount > 0" class="cart-badge">{{ cartItemCount }}</span>
          </div>
        </div>
      </div>

      <!-- Dashboard Title -->
      <h1 class="dashboard-title">Cafe Beata</h1>

      <!-- Display category title dynamically -->
      <div class="category-header">
        <h2>{{ isFirstTimeUser || currentCategory === 'All' ? 'All Menu Items' : (currentCategory === 'All Drinks' ? 'Menu' : currentCategory) }}</h2>
      </div>

      <!-- Loading indicator -->
      <div v-if="isLoading" class="loading-indicator">
        <p>Loading menu items...</p>
      </div>

      <!-- Display filtered items based on the current category -->
      <div v-else-if="filteredItems.length" class="items">
        <div
          v-for="item in filteredItems"
          :key="item.id || item.name"
          class="item"
          @click="checkAndNavigate(item)"
          :class="{
            'out-of-stock': !itemStocks[item.id]?.quantity
          }"
        >
          <img :src="getImagePath(item.image)" :alt="item.name" />
          <div class="item-details">
            <span>{{ item.name }}</span>
            <span class="item-price">₱{{ Number(item.price).toFixed(2) }}</span>
          </div>
        </div>
      </div>
      <div v-else class="no-items">
        <p>No items found in this category.</p>
      </div>

      <!-- Item Click Modal -->
      <div v-if="showItemModal" class="item-modal">
        <div class="modal-content">
          <span class="close" @click="closeItemModal">&times;</span>
          <div class="modal-item-details">
            <img :src="selectedItem ? getImagePath(selectedItem.image) : require('@/assets/default.png')" :alt="selectedItem ? selectedItem.name : ''" />
            <h3>{{ selectedItem ? selectedItem.name : '' }}</h3>
            <p class="price">₱{{ selectedItem ? Number(selectedItem.price).toFixed(2) : '0.00' }}</p>
            
            <!-- Add quantity controls -->
            <div class="modal-quantity-controls">
              <button @click="decreaseModalQuantity" class="quantity-btn">-</button>
              <span class="quantity-display">{{ modalQuantity }}</span>
              <button @click="increaseModalQuantity" class="quantity-btn">+</button>
            </div>
            <p class="total-price">Total: ₱{{ selectedItem ? (Number(selectedItem.price) * modalQuantity).toFixed(2) : '0.00' }}</p>
          </div>
          <div class="modal-buttons">
            <button @click="addToCart" class="add-cart-btn">Add to Cart</button>
            <button @click="orderNow" class="order-now-btn">Order Now</button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>



<script>
import { eventBus } from "@/utils/eventBus"; // Correct the path if needed

export default {
  components: {
  },
  data() {
    return {
      userName: '',
      userProfileImage: '',
      userEmail: localStorage.getItem('userEmail'),
      unreadNotificationsCount: 0,   
      refreshInterval: null,  
      itemsRefreshInterval: null,
      searchQuery: '',
      isDarkMode: localStorage.getItem("darkMode") === "enabled",
      currentCategory: 'All Drinks',
      currentTime: new Date().toLocaleTimeString(),
      isSidebarOpen: localStorage.getItem('sidebarOpen') === 'true',
      isLoading: false,
      apiItems: [],
      filteredItems: [],
      categories: [],
      showItemModal: false,
      selectedItem: null,
      cartItemCount: 0,
      cart: [], // Array to store cart items
      modalQuantity: 1,
      itemStocks: {}, // Add back the itemStocks property
      ws: null,
      wsConnected: false,
      isFirstTimeUser: false, // Add this flag for first time users
    };
  },

  

  created() {
      this.updateNotificationCount();
    window.addEventListener("notificationUpdated", this.updateNotificationCount); // Listen for changes
    window.addEventListener("items-updated", this.handleItemsUpdated); // Listen for item updates
    this.initWebSocket(); // Initialize WebSocket connection
  },
beforeUnmount() {
      window.removeEventListener("notificationUpdated", this.updateNotificationCount);
      window.removeEventListener("items-updated", this.handleItemsUpdated);
      window.removeEventListener('categories-updated', this.handleCategoriesUpdated);
      this.stopPollingForNewNotifications();
      this.stopPollingForItems();
      window.removeEventListener('storage', this.updateCartCount);
      if (this.ws) {
        this.ws.close();
      }
  },
  
  async mounted() {
    this.$watch(
      () => eventBus.notificationsCount,
      (newCount) => {
        this.unreadNotificationsCount = newCount;
      }
    );
    
    // Check if this is a first login
    this.isFirstTimeUser = this.checkFirstLogin();
    
    // Check for last viewed category first
    const lastViewedCategory = localStorage.getItem('lastViewedCategory');
    if (this.isFirstTimeUser) {
      // For first login, set category to 'All' to show all items
      this.currentCategory = 'All';
    } else if (lastViewedCategory) {
      this.currentCategory = lastViewedCategory;
    } else if (this.$route.query.category) {
      // Only use route query if no last viewed category exists
      this.currentCategory = this.$route.query.category;
    }
    
    this.updateTime();
    this.applyDarkMode(this.isDarkMode);
    this.startPollingForNewNotifications();
    await this.loadUserProfile();
    await this.fetchItems();
    this.startPollingForItems();
    await this.loadCategories();
    
    // Filter items after loading everything
    this.filterItems();
    
    window.addEventListener('categories-updated', this.handleCategoriesUpdated);
    this.updateCartCount();
    window.addEventListener('storage', this.updateCartCount);
  },
    
 
  methods: {
    // Check if this is a first login or account creation
    checkFirstLogin() {
      const firstLoginKey = `first_login_${this.userEmail}`;
      const isFirstLogin = localStorage.getItem(firstLoginKey) !== 'false';
      
      // Mark as not first login anymore
      if (isFirstLogin) {
        localStorage.setItem(firstLoginKey, 'false');
      }
      
      return isFirstLogin;
    },
    
    // Handle item updates from ItemEditor
    handleItemsUpdated(event) {
      console.log('Items updated event received:', event.detail);
      
      // Silently refresh items without showing loading indicators
      try {
        fetch('http://localhost:8000/api/items')
          .then(response => response.json())
          .then(data => {
            if (data.items) {
              this.apiItems = data.items;
              this.filterItems();
            }
          });
      } catch (error) {
        console.error('Error refreshing items after update:', error);
      }
      
      // Only switch to the category if the user is not on the "All Drinks" view
      // This way, new items will appear in the All Drinks view without changing the user's context
      if (this.currentCategory !== 'All Drinks' && event.detail.action !== 'deleted' && 
          event.detail.category && event.detail.category !== this.currentCategory) {
        this.currentCategory = event.detail.category;
      }
    },
    
    // New method to manually refresh items
    async refreshItems() {
      this.isLoading = true;
      await this.fetchItems();
      this.filterItems();
      this.isLoading = false;
    },
    
    // New method to start polling for items
    startPollingForItems() {
      this.itemsRefreshInterval = setInterval(async () => {
        // Fetch items silently in the background without showing loading indicators
        try {
          const response = await fetch('http://localhost:8000/api/items');
          const data = await response.json();
          if (data.items) {
            this.apiItems = data.items;
            // Update filtered items without changing the loading state
            this.filterItems();
          }
        } catch (error) {
          console.error('Error fetching items during background refresh:', error);
        }
      }, 10000); // Check for new items every 10 seconds
    },
    
    // New method to stop polling for items
    stopPollingForItems() {
      if (this.itemsRefreshInterval) {
        clearInterval(this.itemsRefreshInterval);
      }
    },
    
    // New method to fetch items from API
    async fetchItems() {
      const wasLoading = this.isLoading;
      
      try {
        const [itemsResponse, stocksResponse] = await Promise.all([
          fetch('http://localhost:8000/api/items'),
          fetch('http://localhost:8000/api/stocks')
        ]);
        
        const itemsData = await itemsResponse.json();
        const stocksData = await stocksResponse.json();
        
        if (itemsData.items) {
          this.apiItems = itemsData.items;
          console.log('Fetched items from API:', this.apiItems);
        }
        
        if (stocksData.success) {
          // Convert array to object for easier lookup
          this.itemStocks = stocksData.items.reduce((acc, stock) => {
            acc[stock.item_id] = stock;
            return acc;
          }, {});
        }
        
        if (!wasLoading) {
          this.filterItems();
        }
      } catch (error) {
        console.error('Error fetching items or stocks:', error);
      } finally {
        if (wasLoading) {
          this.isLoading = false;
        }
      }
    },

    async loadUserProfile() {
      try {
        if (!this.userEmail) return;
        
        const response = await fetch(`http://127.0.0.1:8000/profile/${this.userEmail}`);
        const data = await response.json();
        
        if (response.ok) {
          this.userName = data.username;
          this.userProfileImage = data.avatar ? `http://127.0.0.1:8000${data.avatar}` : require('@/assets/default.png');
        }
      } catch (error) {
        console.error('Error loading profile:', error);
      }
    },


  updateNotificationCount() {
      const userName = localStorage.getItem("userName");
      if (userName) {
        const userNotificationsKey = `user_notifications_${userName}`;
        const notifications = JSON.parse(localStorage.getItem(userNotificationsKey)) || [];
          const unreadCount = notifications.filter(notification => !notification.read).length;
        // Manually trigger a reactivity update for unreadNotificationsCount
        this.unreadNotificationsCount = unreadCount;
  }
  },
    
       startPollingForNewNotifications() {
      this.refreshInterval = setInterval(() => {
        this.updateNotificationCount(); // Check for new notifications
      }, 5000); // Update every 5 seconds (you can adjust this interval)
    },
    
      stopPollingForNewNotifications() {
      clearInterval(this.refreshInterval);
    },

 toggleDarkMode() {
    this.isDarkMode = !this.isDarkMode;
    localStorage.setItem("darkMode", this.isDarkMode ? "enabled" : "disabled");
    this.applyDarkMode(this.isDarkMode);
  },
  applyDarkMode(isDark) {
    if (isDark) {
      document.body.classList.add('dark-mode');
    } else {
      document.body.classList.remove('dark-mode');
    }
  },


 updateTime() {
  setInterval(() => {
    const now = new Date();
    let hours = now.getHours();
    let minutes = now.getMinutes();
    let seconds = now.getSeconds();
    let ampm = hours >= 12 ? 'PM' : 'AM';

    hours = hours % 12 || 12; // Convert 24-hour time to 12-hour format
    minutes = minutes < 10 ? '0' + minutes : minutes; // Ensure two digits for minutes
    seconds = seconds < 10 ? '0' + seconds : seconds; // Ensure two digits for seconds

    this.currentTime = `${hours}:${minutes}:${seconds} ${ampm}`; // Example: 2:03:11 PM
  }, 1000); // Update every second
    },

    filterCategory(category) {
      // If this is a first-time user and they click a category, they're no longer in first-time mode
      if (this.isFirstTimeUser) {
        this.isFirstTimeUser = false;
      }
      
      this.currentCategory = category;
      // Save the selected category
      localStorage.setItem('lastViewedCategory', category);
      this.filterItems();
      
      // Close sidebar on mobile after selecting a category
      if (window.innerWidth <= 768) {
        this.closeSidebar();
      }
    },

    getImagePath(image) {
      // Handle cases where image is undefined or null
      if (!image) {
        return require('@/assets/default.png');
      }

      // If it's already a full URL, return it as is
      if (typeof image === 'string' && image.startsWith('http')) {
        return image;
      }

      // If it's a backend path, prepend the backend URL
      if (typeof image === 'string' && image.startsWith('/uploads')) {
        return `http://localhost:8000${image}`;
      }

      // Try to load from assets
      try {
        return require(`@/assets/${image}`);
      } catch (error) {
        console.error(`Failed to load image: ${image}`, error);
        return require('@/assets/default.png');
      }
    },

    filterItems() {
      const query = this.searchQuery.toLowerCase();
      
      // Filter API items based on category and search query
      if (this.isFirstTimeUser || this.currentCategory === 'All') {
        // For first login or 'All' category, show all items (both drinks and food)
        this.filteredItems = this.apiItems.filter(item => 
          item.name.toLowerCase().includes(query)
        );
      } else if (this.currentCategory === 'All Drinks') {
        // For "All Drinks", get all items that are drink categories
        this.filteredItems = this.apiItems.filter(item => 
          !this.foodCategories.includes(item.category) && // Exclude food categories
          item.name.toLowerCase().includes(query)
        );
      } else if (this.currentCategory === 'All Food') {
        // For "All Food", get all items that are food categories
        this.filteredItems = this.apiItems.filter(item => 
          this.foodCategories.includes(item.category) && // Include only food categories
          item.name.toLowerCase().includes(query)
        );
      } else {
        // For specific categories, filter by category and search query
        this.filteredItems = this.apiItems.filter(item => 
          item.category === this.currentCategory &&
          item.name.toLowerCase().includes(query)
        );
      }
      
      // Log the filtered items for debugging
      console.log(`Filtered ${this.filteredItems.length} items for category: ${this.currentCategory}`);
    },

   checkAndNavigate(item) {
      const stock = this.itemStocks[item.id];
      if (!stock || stock.quantity === 0) {
        alert('Sorry, this item is currently out of stock.');
        return;
      }
      this.showItemModal = true;
      this.selectedItem = item;
    },
    handleLogout() {
      localStorage.removeItem('loggedIn');
      this.$router.push({ name: 'Login' });
    },
    handleOrderHistory() {
      this.$router.push({ name: 'OrderHistory' });
    },
    handleProfile() {
      this.$router.push({ name: 'UserProfileCafe' }); // Navigate to the Profile page
    },
    toggleSidebar() {
      this.isSidebarOpen = !this.isSidebarOpen; // Toggle sidebar open/close
      localStorage.setItem('sidebarOpen', this.isSidebarOpen.toString());
    },
    closeSidebar() {
      this.isSidebarOpen = false;
      localStorage.setItem('sidebarOpen', this.isSidebarOpen.toString());
    },
    getCategoryIcon(category) {
      const foundCategory = this.categories.find(cat => cat.name === category);
      return foundCategory ? foundCategory.icon : 'fas fa-utensils';
    },
    async loadCategories() {
      try {
        const response = await fetch('http://localhost:8000/api/categories');
        const data = await response.json();
        if (data.categories) {
          this.categories = data.categories;
          
          // If current category doesn't exist anymore, reset to a valid one
          const allCategoryNames = [...this.drinkCategories, ...this.foodCategories];
          
          // For first time users, we've already set the category to 'All'
          if (!this.isFirstTimeUser && 
              !allCategoryNames.includes(this.currentCategory) && 
              this.currentCategory !== 'All' &&
              this.currentCategory !== 'All Drinks' && 
              this.currentCategory !== 'All Food') {
            this.currentCategory = this.drinkCategories.length > 0 ? 'All Drinks' : 
                                  (this.foodCategories.length > 0 ? 'All Food' : 'All');
            localStorage.setItem('lastViewedCategory', this.currentCategory);
          }
        }
      } catch (error) {
        console.error('Error loading categories:', error);
      }
    },
    handleCategoriesUpdated() {
      this.loadCategories();
      this.fetchItems();
    },
    goToCart() {
      // Navigate to confirm order page to view cart
      this.$router.push({ name: "ConfirmOrder" });
    },
    closeItemModal() {
      this.showItemModal = false;
      this.selectedItem = null;
      this.modalQuantity = 1; // Reset quantity when closing modal
    },
    async addToCart() {
      if (!this.selectedItem) return;
      
      const stock = this.itemStocks[this.selectedItem.id];
      if (!stock || stock.quantity < this.modalQuantity) {
        alert('Sorry, not enough stock available.');
        return;
      }
      
      const imagePath = this.getImagePath(this.selectedItem.image);
      const userCartKey = `cart_${this.userName}`;
      let cart = JSON.parse(localStorage.getItem(userCartKey)) || [];
      
      const existingItemIndex = cart.findIndex(item => item.name === this.selectedItem.name);
      
      if (existingItemIndex !== -1) {
        const newQuantity = cart[existingItemIndex].quantity + this.modalQuantity;
        if (newQuantity > stock.quantity) {
          alert('Sorry, not enough stock available for the requested quantity.');
          return;
        }
        cart[existingItemIndex].quantity = newQuantity;
      } else {
        cart.push({
          id: this.selectedItem.id,
          name: this.selectedItem.name,
          price: this.selectedItem.price,
          image: imagePath,
          quantity: this.modalQuantity
        });
      }
      
      localStorage.setItem(userCartKey, JSON.stringify(cart));
      this.updateCartCount();
      
      // Save current category before closing modal
      localStorage.setItem('lastViewedCategory', this.currentCategory);
      
      this.closeItemModal();
    },
    orderNow() {
      if (!this.selectedItem) return;
      
      const imagePath = this.getImagePath(this.selectedItem.image);
      const userCartKey = `cart_${this.userName}`;
      let cart = JSON.parse(localStorage.getItem(userCartKey)) || [];
      
      const existingItemIndex = cart.findIndex(item => item.name === this.selectedItem.name);
      
      if (existingItemIndex !== -1) {
        cart[existingItemIndex].quantity += this.modalQuantity;
      } else {
        cart.push({
          name: this.selectedItem.name,
          price: this.selectedItem.price,
          image: imagePath,
          quantity: this.modalQuantity
        });
      }
      
      // Save current category before navigating
      localStorage.setItem('lastViewedCategory', this.currentCategory);
      
      localStorage.setItem(userCartKey, JSON.stringify(cart));
      this.updateCartCount();
      
      this.$router.push({
        name: "ConfirmOrder"
      });
    },
    updateCartCount() {
      const userCartKey = `cart_${this.userName}`;
      const cart = JSON.parse(localStorage.getItem(userCartKey)) || [];
      // Count unique items instead of total quantities
      this.cartItemCount = cart.length;
    },
    decreaseModalQuantity() {
      if (this.modalQuantity > 1) {
        this.modalQuantity -= 1;
      }
    },
    increaseModalQuantity() {
      const stock = this.itemStocks[this.selectedItem?.id];
      if (stock && this.modalQuantity < stock.quantity) {
        this.modalQuantity += 1;
      } else {
        alert('Maximum available stock reached.');
      }
    },
    initWebSocket() {
      const wsUrl = `ws://${window.location.hostname}:8000/ws/orders`;
      this.ws = new WebSocket(wsUrl);
      
      this.ws.onopen = () => {
        console.log('WebSocket connected');
        this.wsConnected = true;
        // Initial fetch of data when connection is established
        this.fetchItems();
        this.loadCategories();
      };
      
      this.ws.onmessage = async (event) => {
        try {
          const data = JSON.parse(event.data);
          console.log('WebSocket message received:', data);
          
          if (data.type === 'stock_update') {
            // Update the itemStocks object with the new stock data
            this.itemStocks[data.item_id] = {
              quantity: data.new_quantity,
              min_stock_level: data.min_stock_level
            };
            
            // If the item is currently selected in the modal, update its stock info
            if (this.selectedItem && this.selectedItem.id === data.item_id) {
              this.selectedItem = {
                ...this.selectedItem,
                stock: data.new_quantity
              };
              
              // If the item is out of stock, close the modal
              if (data.new_quantity === 0) {
                this.closeItemModal();
              }
            }
            
            // Update the filtered items if necessary
            this.filteredItems = this.filteredItems.map(item => {
              if (item.id === data.item_id) {
                return {
                  ...item,
                  stock: data.new_quantity
                };
              }
              return item;
            });
          } else if (data.type === 'menu_update') {
            // Refresh items when menu changes
            await this.fetchItems();
            this.filterItems();
          } else if (data.type === 'category_update') {
            console.log('Category update received:', data);
            // Refresh categories and items when categories change
            await this.loadCategories();
            await this.fetchItems();
            this.filterItems();
            
            // If the current category was renamed, update the selection
            if (data.action === 'update' && data.category.old_name === this.currentCategory) {
              this.currentCategory = data.category.name;
              localStorage.setItem('lastViewedCategory', this.currentCategory);
            }
            
            // If the current category was deleted, reset to default
            if (data.action === 'delete' && data.category_name === this.currentCategory) {
              this.currentCategory = this.categories.length > 0 ? 
                (this.drinkCategories.length > 0 ? 'All Drinks' : 
                 (this.foodCategories.length > 0 ? this.foodCategories[0] : 'All')) : 'All';
              localStorage.setItem('lastViewedCategory', this.currentCategory);
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
  },
  watch: {
    isDarkMode: {
      handler(newValue) {
        if (newValue) {
          document.body.classList.add('dark-mode');
        } else {
          document.body.classList.remove('dark-mode');
        }
      },
      immediate: true
    }
  },
  computed: {
    drinkCategories() {
      return this.categories
        .filter(cat => cat.type === 'drinks')
        .map(cat => cat.name);
    },
    
    foodCategories() {
      return this.categories
        .filter(cat => cat.type === 'food')
        .map(cat => cat.name);
    }
  },
};
</script>


<style scoped>
/* Add loading indicator styles */
.loading-indicator {
  text-align: center;
  padding: 20px;
  font-size: 18px;
  color: #d12f7a;
}

/* All Menu Items button styling */
.all-items-button {
  background-color: #f8d1d1;
  margin: 10px 20px;
  border-radius: 10px;
  font-weight: bold;
  color: #d12f7a;
  transition: all 0.3s ease;
}

.all-items-button:hover {
  background-color: #f8c6d0;
  transform: scale(1.02);
}

.dark-mode .all-items-button {
  background-color: #444;
  color: #f8c6d0;
}

.dark-mode .all-items-button:hover {
  background-color: #555;
}

.no-items {
  text-align: center;
  padding: 20px;
  font-size: 18px;
  color: #666;
}

.utility-divider {
  border: none;
  height: 1px;
 background-color: rgba(0, 0, 0, 0.1);
  margin: 10px 0;
}

.user-profile-section {
  padding: 20px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
  margin-bottom: 20px;
}

.profile-image {
  width: 80px;
  height: 80px;
  border-radius: 50%;
  overflow: hidden;
  border: 2px solid rgba(255, 255, 255, 0.2);
}

.profile-image img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.user-name {
  font-size: 1.1rem;
  font-weight: 500;
  color: #222;
}

.theme-toggle {
  position: absolute;
  top: 20px;
  left: 20px;
  background: transparent;
  border: none;
  font-size: 1.2rem;
  cursor: pointer;
  padding: 8px;
  border-radius: 50%;
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s ease;
}

.theme-toggle:hover {
 background-color: rgba(0, 0, 0, 0.05);
}

.theme-toggle .light-icon {
  color: #333;
}

.theme-toggle .dark-icon {
  color: #fff;
}

.light-mode 

.theme-toggle {
  color: #333;
}

.light-mode 
.user-name {
  color: #333;
}
.dark-mode .user-name {
  color: #fff;
}
.dark-mode .theme-toggle {
  color: #fff;
}
.dark-mode .utility-divider {
  background-color: rgba(255, 255, 255, 0.1);
}

.dark-mode .item {
  background-color: #555555; /* Light grey background for dark mode */
  color: #ffffff; /* Light text color */
  box-shadow: 0px 4px 6px rgba(255, 255, 255, 0.1); /* Lighter box shadow */
}

.dark-mode .item span {
  color: #ffffff; /* Light text for item span */
}

.dark-mode .item-price {
  background-color: #6e6e6e; /* Lighter background for price */
  color: #ffffff; /* Light text color for price */
}

.dark-mode .item-price:hover {
  background-color: #888888; /* Darker price background on hover */
  cursor: pointer;
}


.dark-mode-button, 
.notification-button {
  position: relative;
  background-color: rgb(48, 41, 44);
  color: white;
  padding: 8px 12px;
  font-size: 14px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background 0.3s ease-in-out;
}

.notification-badge {
  position: absolute;
  top: -5px;
  right: -5px;
  background-color: red;
  color: white;
  border-radius: 50%; /* This ensures it's fully circular */
  font-size: 12px;
  width: 20px; /* Set a fixed width */
  height: 20px; /* Set a fixed height to ensure it's circular */
  display: flex;
  justify-content: center; /* Centers the content horizontally */
  align-items: center; /* Centers the content vertically */
}

.dark-mode-button i, 
.notification-button i {
  font-size: 18px; /* Adjust icon size */
}

.top-bar-buttons {
  display: flex;
  gap: 10px; /* Space between buttons */
}

.notification-button {
  background-color: rgb(48, 41, 44); /* Same background as dark mode */
  color: white;
  padding: 8px 12px;
  font-size: 14px;
  border-radius: 5px;
  cursor: pointer;
  transition: background 0.3s ease-in-out;
}

.notification-button:hover {
  background-color: #b82d67; /* Same hover effect as dark mode button */
}

.notification-button i {
  font-size: 18px; /* Adjust the icon size */
}

.dark-mode {
  background-color: #121212;
  color: #ffffff;
}

/* Buttons and Sidebar in Dark Mode */
.dark-mode .sidebar,
.dark-mode .dashboard,
.dark-mode .top-bar,
.dark-mode .content {
  background-color: #1e1e1e;
  color: #ffffff;
}


/* Dark Mode Button Styling */
.dark-mode-button {
  background-color: rgb(48, 41, 44);
  color: white;
  padding: 8px 12px;
  font-size: 14px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background 0.3s ease-in-out;
}

.dark-mode .sidebar,
.dark-mode .sidebar-category h3,
.dark-mode .sidebar-category ul li {
  color: #ffffff !important;
}

/* Dark Mode for Live Time */
.dark-mode .live-time {
  color: #ffffff !important;
}


   .item-details {
    display: flex;
    flex-direction: column; /* Stack the name and price vertically */
    align-items: center;
    margin-top: 10px;
  }

  /* Style the price to highlight it */
  .item-price {
    font-size: 18px;
    font-weight: bold;
    color: #d12f7a; /* Pink color for the price */
    transition: all 0.3s ease;
    margin-top: 5px; /* Add some space between the name and price */
  }

  /* Glowing effect on hover */
  .item-price:hover {
    color: #fff; /* White text color on hover */
    text-shadow: 0 0 10px rgba(209, 47, 122, 1), 0 0 20px rgba(209, 47, 122, 0.7); /* Glowing text effect */
  }

  /* Glowing Button Styles */
  .profile-button,
  .order-history-button,
  .logout-button {
    padding: 15px 40px;
    border: none;
    outline: none;
    color: #FFF;
    cursor: pointer;
    position: relative;
    z-index: 0;
    border-radius: 12px;
    background-color: transparent;
    border: 2px solid #d12f7a; /* Adjust border color */
    font-size: 14px;
   
  }

  /* Glowing effect */
  .profile-button::after,
  .order-history-button::after,
  .logout-button::after {
    content: "";
    z-index: -1;
    position: absolute;
    width: 100%;
    height: 100%;
    background-color: #333;
    left: 0;
    top: 0;
    border-radius: 10px;
  }

  /* Animation for glowing */
  .profile-button::before,
  .order-history-button::before,
  .logout-button::before {
    content: "";
    background: linear-gradient(
      45deg,
      #FF0000, #FF7300, #FFFB00, #48FF00,
      #00FFD5, #002BFF, #FF00C8, #FF0000
    );
    position: absolute;
    top: -2px;
    left: -2px;
    background-size: 600%;
    z-index: -1;
    width: calc(100% + 4px);
    height: calc(100% + 4px);
    filter: blur(8px);
    animation: glowing 20s linear infinite;
    transition: opacity .3s ease-in-out;
    border-radius: 10px;
    opacity: 0;
  }

  /* Hover effect for glowing */
  .profile-button:hover::before,
  .order-history-button:hover::before,
  .logout-button:hover::before {
    opacity: 1;
  }

  /* Active button state */
  .profile-button:active:after,
  .order-history-button:active:after,
  .logout-button:active:after {
    background: transparent;
  }

  .profile-button:active,
  .order-history-button:active,
  .logout-button:active {
    color: #000;
    font-weight: bold;
    background-color: #d12f7a; /* Active background color */
    border-color: #d12f7a; /* Border color */
  }

  /* Glow Animation */
  @keyframes glowing {
    0% {background-position: 0 0;}
    50% {background-position: 400% 0;}
    100% {background-position: 0 0;}
  }

/* Existing styles (unchanged) */
.profile-button {
  background-color: rgb(100, 14, 51);
  color: white;
  padding: 10px;
  font-size: 18px;
  border: none;
  border-radius: 50%;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.3s ease-in-out;
  width: 40px; /* Adjust width and height for round button */
  height: 40px; /* Adjust width and height for round button */
}

.profile-button i {
  font-size: 24px; /* Adjust icon size */
}

.profile-button:hover {
  background-color: #b82d67;

}

.dashboard {
  display: flex;
  min-height: 100vh;
  flex-direction: column;
  background-color: #fce6e6;
  width: 100%;
  overflow-x: hidden;
  position: relative;
}

html, body {
  min-height: 100vh;
  margin: 0;
  padding: 0;
  background-color: #fce6e6;
}

/* Sidebar */
.sidebar {
  position: fixed;
  top: 0;
  left: -280px;
  height: 100vh;
  width: 280px;
  background-color: white;
  transition: all 0.3s ease;
  z-index: 1000;
  display: flex;
  flex-direction: column;
  padding: 20px 0 20px 0; /* Add bottom padding */
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
}

.sidebar.light-mode {
  background-color: #fce6e6;
}

.sidebar.dark-mode {
  background-color: #333;
  color: #fff;
}

.sidebar.dark-mode .utility-button,
.sidebar.dark-mode .menu-item,
.sidebar.dark-mode .profile-section,
.sidebar.dark-mode h3 {
  color: #fff;
}

.sidebar.dark-mode .utility-button:hover,
.sidebar.dark-mode .menu-item:hover {
  background-color: rgba(255, 255, 255, 0.1);
}

.sidebar.dark-mode .profile-section,
.sidebar.dark-mode .utility-section,
.sidebar.dark-mode .logout {
  border-color: rgba(255, 255, 255, 0.1);
}

.sidebar.open {
  left: 0;
}

.close-sidebar {
  position: absolute;
  top: 15px;
  right: 15px;
  background: none;
  border: none;
  font-size: 18px;
  cursor: pointer;
  color: inherit;
  padding: 5px;
}

.profile-section {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 15px 20px;
  color: inherit;
  font-size: 15px;
  border-bottom: 1px solid rgba(0, 0, 0, 0.1);
  background: none;
  border: none;
  width: 100%;
  cursor: pointer;
  text-align: left;
}

.profile-section:hover {
  background-color: rgba(0, 0, 0, 0.05);
}

.utility-section {
  padding: 10px 0;
  border-bottom: 1px solid rgba(0, 0, 0, 0.1);
}

.utility-button {
  width: 100%;
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 12px 20px;
  background: none;
  border: none;
  cursor: pointer;
  color: inherit;
  font-size: 15px;
  text-decoration: none;
  transition: background-color 0.2s;
}

.utility-button:hover {
  background-color: rgba(0, 0, 0, 0.05);
}

.notification-link {
  position: relative;
  text-decoration: none;
  color: inherit;
}

.notification-icon {
  position: relative;
  display: inline-block;
  width: 20px;
}

.notification-badge {
  position: absolute;
  top: -8px;
  right: -8px;
  background-color: red;
  color: white;
  border-radius: 50%; /* This ensures it's fully circular */
  font-size: 11px;
  min-width: 15px;
  height: 15px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.menu-section {
  flex: 1;
  padding: 15px 0 0 0;
  overflow-y: auto;
  margin-bottom: 0;
}

.menu-section h3 {
  padding: 0 20px;
  margin: 10px 0 5px;
  font-size: 16px;
  color: inherit;
  font-weight: 500;
}

.menu-items {
  display: flex;
  flex-direction: column;
  margin-bottom: 15px;
}

.menu-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 8px 20px;
  background: none;
  border: none;
  cursor: pointer;
  color: inherit;
  font-size: 15px;
  text-align: left;
  transition: background-color 0.2s;
}

.menu-item:hover {
  background-color: rgba(0, 0, 0, 0.05);
}

.menu-item i {
  width: 20px;
  text-align: center;
  font-size: 14px;
}

.logout-container {
  border-top: 1px solid rgba(0, 0, 0, 0.1);
  padding: 0;
  background-color: #fce6e6;
  margin-bottom: 20px; /* Add space after logout button */
}

.dark-mode .logout-container {
  background-color: #333;
  border-top-color: rgba(255, 255, 255, 0.1);
}

.logout {
  color: inherit;
  padding: 10px 20px;
  width: 100%;
  display: flex;
  align-items: center;
  gap: 10px;
}

/* Mobile Responsive */
@media (max-width: 768px) {
  .content {
    margin-left: 0;
    padding: 15px;
  }
  
  .menu-button {
    display: block;
  }
}

/* Desktop Responsive */
@media (min-width: 769px) {
  .content {
    margin-left: 0; /* Remove default margin to match mobile behavior */
  }
  
  .menu-button {
    display: block; /* Show menu button on desktop */
    background: rgb(255, 255, 255);
    color: black;
    padding: 10px 15px;
    font-size: 18px;
    border-radius: 15px;
    transition: background 0.3s ease-in-out;
  }

  /* When sidebar is open */
  .sidebar.open + .content {
    margin-left: 270px;
  }
}

/* Update menu button base styles */
.menu-button {
  position: fixed;
  top: 15px;
  left: 15px;
  z-index: 300;
  background: rgb(255, 255, 255);
  color: black;
  padding: 10px 15px;
  font-size: 18px;
  border: none;
  border-radius: 15px;
  cursor: pointer;
  transition: background 0.3s ease-in-out;
  display: block; /* Always show the menu button */
}

.menu-icon-container {
  position: relative;
  display: inline-block;
}

.menu-notification-badge {
  position: absolute;
  top: -8px;
  right: -20px;
  background-color: red;
  color: white;
  border-radius: 50%;
  font-size: 12px;
  min-width: 18px;
  height: 18px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
}

/* Add overlay styles */
.overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.5);
  z-index: 299;
  display: none;
}

.overlay.show {
  display: block;
}

/* Update content transition */
.content {
  transition: margin-left 0.3s ease;
}

/* Style the container for both logo and live time */
.logo-time-container {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  width: 100%;
}

.search-container { 
  display: flex;
  justify-content: center; 
  padding: 10px;
  border-radius: 15px;
  width: 100%;
} 

/* Style for the live time text */
.live-time {
  font-weight: bold;
  font-size: 15px;
  color: #333;
  margin: 0;
  padding: 0;
  display: flex;
  align-items: center;
}

.live-time p {
  margin: 0;
}

/* Add other necessary styling if needed */
.top-bar {
  display: flex;
  border-radius: 50px;
  background-color: rgb(255, 239, 239);
  flex-direction: column;
  justify-content: center;
  align-items: center;
  padding: 15px 20px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  gap: 15px;
}

.logo-container {
  display: flex;
  align-items: center;
  justify-content: center;
}

.logo-container img {
  width: 80px;
  height: auto;
}

.search-cart-container {
  display: flex;
  align-items: center;
  justify-content: center;
  position: relative;
  width: 100%;
  max-width: 400px;
}

.search-container input {
  width: 100%;
  max-width: 300px;
  padding: 10px 15px;
  border-radius: 20px;
  border: 1px solid #ccc;
  font-size: 14px;
}

/* Live Time text styling */
.live-time {
  font-weight: bold;
  font-size: 15px;
  color: #333;
  margin: 0;
  padding: 0;
  display: flex;
  align-items: center;
}

.live-time p {
  margin: 0;
}

/* Logo container */
.logo-container {
  display: flex;
  align-items: center;
  justify-content: center;
}

.logo-container img {
  width: 80px;
  height: auto;
}

/* Cart icon positioning */
.cart-icon-container {
  position: absolute;
  right: -40px;
  cursor: pointer;
  padding: 10px;
  font-size: 24px;
  color: #d12f7a;
  transition: color 0.3s ease;
}

/* Responsive adjustments */
@media (max-width: 768px) {
  .search-cart-container {
    width: 90%;
  }
  
  .cart-icon-container {
    right: -35px;
    font-size: 20px;
  }
}

@media (min-width: 769px) {
  .top-bar {
    padding: 20px;
  }
  
  .search-container {
    max-width: 400px;
  }
}

/* Dashboard Title */
.dashboard-title {
   font-size: 40px;
  font-weight: bold;
   color: #d12f7a;
  margin-top: 15px;
  margin-bottom: 15px;
  text-align: center;
  font-style: italic; 
  font-family: "Merriweather", serif; 
  letter-spacing: 1px; 
 
}

 .dashboard-title:hover {
    color: #fff; 
    text-shadow: 0 0 10px rgba(209, 47, 122, 1), 0 0 20px rgba(209, 47, 122, 0.7); 

 }
 
 .logo-container {
  display: flex;
  align-items: center;
  justify-content: center;
}

/* Live Time container */
.live-time-container {
  display: flex;
  justify-content: center;
  align-items: center;
  color: #fff;
  font-weight: bold;
  font-size: 20px;
}

/* Style the search bar */
.search-container input {
  border-radius: 20px;
  border: 1px solid #ccc;
  width: 100%;
  max-width: 300px;
}

.order-history-button,
  .logout-button {
    font-size: 12px;
    padding: 6px 10px;
  }
/* Categories Section */
.categories {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.category {
  width: 48%;
}

.category h2 {
  font-size: 24px;
  font-weight: bold;
  color: #d12f7a;
  text-align: center;
  margin-top: 20px;
  margin-bottom: 10px;
}

/* Items Section */
.items {
  display: grid;
    grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
   gap: 20px;
  padding: 20px;
  justify-content: center;
  align-items: center;
}
@media (min-width: 1024px) {
  .items {
    grid-template-columns: repeat(4, 1fr);
  }
}

/* Adjust for smaller screens */
@media (max-width: 1023px) {
  .items {
    grid-template-columns: repeat(3, 1fr);
  }
}

@media (max-width: 768px) {
  .items {
     display: flex;
    grid-template-columns: repeat(2, 1fr);
       flex-direction: column;
    align-items: center;
  }
}

@media (max-width: 480px) {
  .items {
    grid-template-columns: 1fr;
  }
}

/* Item styling */
.item {
  text-align: center;
  background-color: #f8d1d1;
  border-radius: 15px;
  padding: 15px;
  cursor: pointer;
  transition: transform 0.3s ease, opacity 0.3s ease;
  height: auto;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  width: 100%;
  max-width: 200px;
  margin: 0 auto;
  position: relative;
  overflow: hidden;
   box-shadow: 0px 4px 6px rgba(0, 0, 0, 0.1);
}

.item img {
  width: 100px;
  height: 100px;
  object-fit: contain;
}

.item:hover img {
  transform: scale(1.05);
}

.item span {
  font-weight: bold;
  color: #333;
  font-size: 16px;
  text-align: center;
  display: block;
  line-height: 1.3;
  margin-top: 8px;
}

/* Responsive Text Adjustments */
@media (max-width: 768px) {
  .dashboard-title {
    font-size: 35px;
  }

}
 .item-details {
    display: flex;
    justify-content: space-between;
   flex-direction: column;
  align-items: center;
  margin-top: 10px;
  }

  .item-price {
    font-size: 18px;
    font-weight: bold;
    color: #d12f7a; 
    background-color: #f8e1e6; 
    padding: 5px 10px;
    border-radius: 5px;
    margin-top: 5px;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  }

  .item-price:hover {
    background-color: #f8c6d0; 
    cursor: pointer;
  }

  .category h2 {
    font-size: 20px;
  }

  .item span {
    font-size: 14px;
  }

 

@media (max-width: 480px) {
  .dashboard-title {
    font-size: 30px;
  }

  .category h2 {
    font-size: 18px;
  }


  .order-history-button,
  .logout-button {
     font-size: 10px;
    padding: 5px 8px;
  }
}

.category-header {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
  margin-bottom: 20px;
}

.category-header h2 {
  font-size: 24px;
  color:rgb(0, 0, 0);
  text-align: center;
  margin: 0;
  padding: 0;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 1px;
}

/* Dark mode styles for category header */
.dark-mode .category-header h2 {
  color: #f8c6d0;
}

/* Remove refresh button styles that are no longer needed */
.refresh-button,
.refresh-button:hover,
.refresh-button:disabled,
.fa-spin,
@keyframes fa-spin {
  /* These styles will be removed */
}

.search-cart-container {
  display: flex;
  align-items: center;
  width: 80%;
  max-width: 400px;
  position: relative;
}

.search-container { 
  display: flex;
  justify-content: center; 
  padding: 10px;
  border-radius: 15px;
  width: 100%;
  max-width: 300px;
} 

.cart-icon-container {
  position: absolute;
  right: -40px;
  cursor: pointer;
  padding: 10px;
  font-size: 24px;
  color: #d12f7a;
  transition: color 0.3s ease;
}

@media (max-width: 768px) {
  .search-cart-container {
    width: 90%;
    max-width: none;
    position: relative;
  }
  
  .search-container {
    width: 100%;
    max-width: none;
  }
  
  .cart-icon-container {
    right: -35px;
    font-size: 20px;
  }
  
  .search-container input {
    width: 100%;
    max-width: none;
  }
}

.cart-badge {
  position: absolute;
  top: -5px;
  right: -5px;
  background-color: red;
  color: white;
  border-radius: 50%;
  padding: 2px 6px;
  font-size: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.item-modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.modal-content {
  background-color: #fce6e6;
  padding: 30px;
  border-radius: 15px;
  text-align: center;
  width: 90%;
  max-width: 400px;
  position: relative;
}

/* Update close button styles */
.modal-content .close {
  position: absolute;
  top: 10px;
  right: 15px;
  font-size: 32px;
  color: #333;
  cursor: pointer;
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  transition: all 0.3s ease;
  background: none;
  border: none;
  padding: 0;
  line-height: 1;
}

.modal-content .close:hover {
  color: #d12f7a;
  transform: scale(1.1);
}

/* Dark mode styles for close button */
.dark-mode .modal-content .close {
  color: #fff;
}

.dark-mode .modal-content .close:hover {
  color: #f8c6d0;
}

@media (max-width: 768px) {
  .modal-content .close {
    font-size: 28px;
    width: 35px;
    height: 35px;
  }
}

.modal-item-details {
  margin-bottom: 20px;
}

.modal-item-details img {
  width: 150px;
  height: 150px;
  object-fit: contain;
  margin-bottom: 15px;
}

.modal-item-details h3 {
  color: #333;
  margin: 10px 0;
}

.modal-item-details .price {
  color: #d12f7a;
  font-size: 20px;
  font-weight: bold;
}

.modal-buttons {
  display: flex;
  justify-content: center;
  gap: 15px;
}

.add-cart-btn, .order-now-btn {
  padding: 10px 20px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-weight: bold;
  transition: all 0.3s ease;
}

.add-cart-btn {
  background-color: #333;
  color: white;
}

.order-now-btn {
  background-color: #d12f7a;
  color: white;
}

.add-cart-btn:hover {
  background-color: #444;
}

.order-now-btn:hover {
  background-color: #b82d67;
}

/* Dark mode styles */
.dark-mode .modal-content {
  background-color: #333;
  color: white;
}

.dark-mode .modal-item-details h3 {
  color: white;
}

.dark-mode .cart-icon-container {
  color: #f8c6d0;
}

.dark-mode .cart-icon-container:hover {
  color: #f8a1b2;
}

@media (max-width: 768px) {
  .search-cart-container {
    max-width: 300px;
  }
  
  .cart-icon-container {
    font-size: 20px;
  }
  
  .modal-content {
    width: 85%;
    padding: 20px;
  }
  
  .modal-item-details img {
    width: 120px;
    height: 120px;
  }
}

/* Add this to your existing styles */
.added-to-cart-notification {
  position: fixed;
  top: 20px;
  right: 20px;
  background-color: #4CAF50;
  color: white;
  padding: 10px 20px;
  border-radius: 5px;
  z-index: 2000;
  animation: slideIn 0.3s ease-out, fadeOut 0.3s ease-out 0.7s;
}

@keyframes slideIn {
  from {
    transform: translateX(100%);
    opacity: 0;
  }
  to {
    transform: translateX(0);
    opacity: 1;
  }
}

@keyframes fadeOut {
  from {
    opacity: 1;
  }
  to {
    opacity: 0;
  }
}

/* Dark mode support */
.dark-mode .added-to-cart-notification {
  background-color: #45a049;
}

/* Modal Quantity Controls */
.modal-quantity-controls {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 15px;
  margin: 20px 0;
}

.quantity-btn {
  width: 35px;
  height: 35px;
  border: none;
  border-radius: 50%;
  background-color: #d12f7a;
  color: white;
  font-size: 20px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s ease;
}

.quantity-btn:hover {
  background-color: #b82d67;
  transform: scale(1.1);
}

.quantity-display {
  font-size: 24px;
  font-weight: bold;
  color: #333;
  min-width: 40px;
  text-align: center;
}

.total-price {
  font-size: 20px;
  font-weight: bold;
  color: #d12f7a;
  margin-top: 10px;
}

/* Dark mode styles for quantity controls */
.dark-mode .quantity-display {
  color: #fff;
}

.dark-mode .quantity-btn {
  background-color: #444;
}

.dark-mode .quantity-btn:hover {
  background-color: #555;
}

/* Remove all out-of-stock related styles */
.item.out-of-stock {
  opacity: 0.7;
  pointer-events: none;
  position: relative;
}

.item.out-of-stock::after {
  content: "Out of Stock";
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background-color: rgba(244, 67, 54, 0.9);
  color: white;
  padding: 8px 16px;
  border-radius: 4px;
  font-weight: bold;
  z-index: 2;
}

.item.out-of-stock img {
  filter: grayscale(1);
}
</style>
