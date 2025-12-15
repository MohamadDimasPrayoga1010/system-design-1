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

