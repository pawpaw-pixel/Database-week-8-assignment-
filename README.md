-- Create database
CREATE DATABASE IF NOT EXISTS shopdb
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

USE shopdb;

-- Customers table (one-to-many: Customer -> Orders)
CREATE TABLE customers (
  customer_id INT AUTO_INCREMENT PRIMARY KEY,
  first_name VARCHAR(100) NOT NULL,
  last_name VARCHAR(100) NOT NULL,
  email VARCHAR(255) NOT NULL UNIQUE,
  phone VARCHAR(30),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- Addresses (one customer can have many addresses)
CREATE TABLE addresses (
  address_id INT AUTO_INCREMENT PRIMARY KEY,
  customer_id INT NOT NULL,
  address_line1 VARCHAR(255) NOT NULL,
  address_line2 VARCHAR(255),
  city VARCHAR(100) NOT NULL,
  state VARCHAR(100),
  postal_code VARCHAR(20),
  country VARCHAR(100) NOT NULL,
  is_default BOOLEAN DEFAULT FALSE,
  FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
    ON DELETE CASCADE ON UPDATE CASCADE
) ENGINE=InnoDB;

-- Products
CREATE TABLE products (
  product_id INT AUTO_INCREMENT PRIMARY KEY,
  sku VARCHAR(50) NOT NULL UNIQUE,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  price DECIMAL(10,2) NOT NULL,
  stock INT NOT NULL DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- Categories (for many-to-many with products)
CREATE TABLE categories (
  category_id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100) NOT NULL UNIQUE,
  description VARCHAR(255)
) ENGINE=InnoDB;

-- Product ↔ Category (many-to-many)
CREATE TABLE product_categories (
  product_id INT NOT NULL,
  category_id INT NOT NULL,
  PRIMARY KEY (product_id, category_id),
  FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE,
  FOREIGN KEY (category_id) REFERENCES categories(category_id) ON DELETE CASCADE
) ENGINE=InnoDB;

-- Orders (one-to-many: Order -> OrderItems; many orders per customer)
CREATE TABLE orders (
  order_id INT AUTO_INCREMENT PRIMARY KEY,
  customer_id INT NOT NULL,
  order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  status ENUM('pending','paid','shipped','cancelled','refunded') DEFAULT 'pending',
  total_amount DECIMAL(10,2) NOT NULL DEFAULT 0.00,
  shipping_address_id INT,
  billing_address_id INT,
  FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE RESTRICT,
  FOREIGN KEY (shipping_address_id) REFERENCES addresses(address_id) ON DELETE SET NULL,
  FOREIGN KEY (billing_address_id)  REFERENCES addresses(address_id) ON DELETE SET NULL
) ENGINE=InnoDB;

-- OrderItems (composite PK ensures no duplicate same product row per order)
CREATE TABLE order_items (
  order_id INT NOT NULL,
  product_id INT NOT NULL,
  quantity INT NOT NULL CHECK (quantity > 0),
  unit_price DECIMAL(10,2) NOT NULL,
  PRIMARY KEY (order_id, product_id),
  FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE,
  FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE RESTRICT
) ENGINE=InnoDB;

-- Payments (1-to-1 or 1-to-many depending on payment splits)
CREATE TABLE payments (
  payment_id INT AUTO_INCREMENT PRIMARY KEY,
  order_id INT NOT NULL,
  amount DECIMAL(10,2) NOT NULL,
  payment_method ENUM('card','paypal','bank_transfer','cash') NOT NULL,
  paid_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  transaction_ref VARCHAR(255),
  FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE
) ENGINE=InnoDB;

-- Product reviews (customer -> many reviews; product -> many reviews)
CREATE TABLE reviews (
  review_id INT AUTO_INCREMENT PRIMARY KEY,
  product_id INT NOT NULL,
  customer_id INT,
  rating TINYINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
  title VARCHAR(200),
  body TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (product_id) REFERENCES products(product_id) ON DELETE CASCADE,
  FOREIGN KEY (customer_id) REFERENCES customers(customer_id) ON DELETE SET NULL
) ENGINE=InnoDB;

-- Simple users table for admin/auth (example of one-to-one if desired)
CREATE TABLE users (
  user_id INT AUTO_INCREMENT PRIMARY KEY,
  username VARCHAR(100) NOT NULL UNIQUE,
  password_hash VARCHAR(255) NOT NULL,
  role ENUM('admin','manager','support') DEFAULT 'support',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB;

-- Indexes for common queries
CREATE INDEX idx_products_price ON products(price);
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orderitems_product ON order_items(product_id);
-- Customers
INSERT INTO customers(first_name,last_name,email,phone)
VALUES
('John','Doe','john@example.com','+254700111222'),
('Jane','Smith','jane@example.com','+254700333444'),
('Emily','Clark','emily@example.com','+254700555666');

-- Addresses
INSERT INTO addresses(customer_id,address_line1,city,country,is_default)
VALUES
(1,'12 Riverside Ave','Nairobi','Kenya',TRUE),
(2,'88 Market St','Mombasa','Kenya',TRUE),
(3,'7 High St','Nakuru','Kenya',TRUE);

-- Categories
INSERT INTO categories(name,description) VALUES
('Electronics','Computers and devices'),
('Accessories','Peripheral devices and accessories'),
('Phones','Mobile phones');

-- Products
INSERT INTO products(sku,name,description,price,stock) VALUES
('LAP-001','Laptop Pro 15','Powerful laptop',1200.00,10),
('MSE-001','Wireless Mouse','Ergonomic mouse',25.00,150),
('TAB-001','Tablet X','Lightweight tablet',350.00,20),
('KEY-001','Mechanical Keyboard','Tactile keyboard',80.00,30),
('PHN-001','Smartphone Z','Latest smartphone',600.00,50);

-- Product categories mapping
INSERT INTO product_categories(product_id,category_id) VALUES
(1,1), -- Laptop -> Electronics
(2,2), -- Mouse -> Accessories
(3,1),
(3,2),
(4,2),
(5,3);

-- Create an order with items
INSERT INTO orders(customer_id,order_date,status,total_amount,shipping_address_id,billing_address_id)
VALUES (1, NOW(), 'pending', 0.00, 1, 1);
SET @last_order_id = LAST_INSERT_ID();

INSERT INTO order_items(order_id,product_id,quantity,unit_price) VALUES
(@last_order_id, 1, 2, 1200.00),
(@last_order_id, 2, 1, 25.00);

-- Update order total
UPDATE orders o
JOIN (
  SELECT order_id, SUM(quantity * unit_price) AS total
  FROM order_items
  WHERE order_id = @last_order_id
  GROUP BY order_id
) t ON o.order_id = t.order_id
SET o.total_amount = t.total;
