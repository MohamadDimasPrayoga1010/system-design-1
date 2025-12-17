# System Design Cart

Cart:

- Add item
    - Limit until out of stock
- Remove Item
- Update qty
- Remove cart / checkout
- Consistent / multi device (across sessions) per account
    - Phone A (clothes merk A 9)
    - Phone B (clothes merk A 9)
- Race condition(only 1 cart can be checkout)
- 1 day 1 mio add to cart
- Expire 1 month
- Data integrity
    - No price manipulation
    - Parameter tampering
- Detail how checkout work
    - Make sure stock available


## Sequence Diagram

### ADD ITEM

``` mermaid

sequenceDiagram
    participant U as User/Client
    participant API as API Gateway
    participant RL as Rate Limiter
    participant CS as Cart Service
    participant Cache as Redis Cache
    participant Lock as Redis Lock
    participant DB as PostgreSQL
    participant PS as Product Service

    Note over U,PS: SEQUENCE 1: ADD ITEM TO CART
    Note over U,PS: (Rate Limit 1M/day + Stock Validation + Price Integrity)
    
    U->>API: POST /cart/items<br/>{product_id, quantity}
    API->>RL: Check rate limit
    RL->>RL: Increment counter<br/>ratelimit:user:{id}:{date}
    
    alt Rate limit exceeded (> 1M/day)
        RL-->>U: 429 Too Many Requests
    else Within limit
        RL->>CS: Forward request
        CS->>PS: GET /products/{product_id}
        PS-->>CS: {price, stock, name}
        
        alt Stock insufficient
            CS-->>U: 400 Out of Stock
        else Stock available
            CS->>Lock: Try acquire lock<br/>stock:lock:{product_id}
            Lock-->>CS: Lock acquired
            
            CS->>DB: BEGIN TRANSACTION
            CS->>DB: SELECT cart WHERE user_id<br/>AND status='active'
            
            alt Cart not exist
                CS->>DB: INSERT INTO carts<br/>(user_id, expires_at)
            end
            
            CS->>DB: SELECT stock FROM products<br/>WHERE id = product_id<br/>FOR UPDATE
            
            alt Stock still available
                CS->>DB: INSERT INTO cart_items<br/>(cart_id, product_id, quantity,<br/>price_snapshot)<br/>ON CONFLICT UPDATE quantity
                CS->>DB: COMMIT
                CS->>Cache: DELETE cart:{user_id}
                CS->>Lock: Release lock
                CS-->>U: 200 OK<br/>{cart_item_id, message}
            else Stock taken
                CS->>DB: ROLLBACK
                CS->>Lock: Release lock
                CS-->>U: 400 Stock unavailable
            end
        end
    end


```

### REMOVE ITEM FROM CART
```mermaid
sequenceDiagram
    participant U as User/Client
    participant API as API Gateway
    participant CS as Cart Service
    participant Cache as Redis Cache
    participant DB as PostgreSQL

    Note over U,DB: SEQUENCE 2: REMOVE ITEM FROM CART
    
    U->>API: DELETE /cart/items/{cart_item_id}
    API->>CS: Forward request
    
    CS->>DB: SELECT cart_items ci<br/>JOIN carts c ON ci.cart_id = c.cart_id<br/>WHERE ci.id = cart_item_id<br/>AND c.user_id = current_user_id<br/>AND c.status = 'active'
    
    alt Item not found or not owned by user
        DB-->>CS: 0 rows
        CS-->>U: 404 Not Found<br/>{error: "Item not found"}
    else Item found
        DB-->>CS: cart_item data
        
        CS->>DB: BEGIN TRANSACTION
        CS->>DB: DELETE FROM cart_items<br/>WHERE id = cart_item_id
        
        CS->>DB: SELECT COUNT(*) FROM cart_items<br/>WHERE cart_id = cart_id
        
        alt No items left in cart
            CS->>DB: UPDATE carts<br/>SET status = 'empty'<br/>WHERE id = cart_id
        end
        
        CS->>DB: COMMIT
        CS->>Cache: DELETE cart:{user_id}
        CS-->>U: 200 OK<br/>{message: "Item removed"}
    end

```

### AUTO EXPIRE CART

```mermaid
sequenceDiagram
    participant Cron as Cron Scheduler
    participant CS as Cart Service
    participant DB as PostgreSQL
    participant Cache as Redis Cache
    participant Log as Logging Service

    Note over Cron,Log: SEQUENCE 7: AUTO EXPIRE CART (1 Month)
    Note over Cron,Log: Background Job - Runs every hour
    
    loop Every 1 hour
        Cron->>CS: Trigger expire job
        
        CS->>DB: BEGIN TRANSACTION
        
        CS->>DB: SELECT cart_id, user_id<br/>FROM carts<br/>WHERE status = 'active'<br/>AND expires_at < NOW()<br/>LIMIT 1000<br/>FOR UPDATE SKIP LOCKED
        
        alt No expired carts
            DB-->>CS: 0 rows
            CS->>DB: COMMIT
            CS->>Log: INFO: No carts to expire
        else Found expired carts
            DB-->>CS: [{cart_id, user_id}, ...]
            
            CS->>Log: INFO: Found {count} expired carts
            
            loop For each expired cart
                CS->>DB: UPDATE carts<br/>SET status = 'expired',<br/>expired_at = NOW()<br/>WHERE cart_id = cart_id
                
                Note over CS: Keep cart_items for analytics<br/>Don't delete, just mark cart as expired
                
                CS->>DB: DELETE FROM stock_reservations<br/>WHERE cart_id = cart_id
                
                CS->>Cache: DELETE cart:{user_id}
                
                CS->>Log: INFO: Expired cart {cart_id}<br/>for user {user_id}
            end
            
            CS->>DB: COMMIT
            
            CS->>Log: INFO: Expired {count} carts successfully
        end
    end
    
    Note over Cron,Log: Optional: Notification to users
    opt Send notification
        CS->>CS: Get users with expired carts
        CS->>CS: Send "Cart expired" notification
        Note over CS: (via notification service - out of scope)
    end

```
