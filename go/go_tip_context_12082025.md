6. Use Context for Cancellations and Timeouts
Senior engineers design for reliability. If your API calls or DB queries don’t use context.Context, you’re asking for trouble.

    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()
    if err := service.Process(ctx, orderID); err != nil {
        log.Fatal(err)
    }

This makes your services predictable under load and prevents resource leaks.

# What context gives you
Cancellation: stop work early when caller doesn’t care anymore.

Deadline/Timeout: bound slow calls so they don’t hang.

Propagation: one cancel stops everything down the call chain.

Leak prevention: goroutines exit when ctx.Done() closes.


Practical examples
1) HTTP handler → service → repository
    
    type Handler struct {
            svc *Service
        }

    type Service struct {
            repo *Repo
        }

    func (h *Handler) CreateOrder(w http.ResponseWriter, r *http.Request) {
        ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
        defer cancel()

        if err := h.svc.CreateOrder(ctx, parseOrder(r)); err != nil {
            http.Error(w, err.Error(), http.StatusGatewayTimeout) // or map err properly
            return
        }
        w.WriteHeader(http.StatusCreated)
    }

    func (s *Service) CreateOrder(ctx context.Context, o Order) error {
        // do stuff, then persist
        return s.repo.Save(ctx, o)
    }

    func (r *Repo) Save(ctx context.Context, o Order) error {
        // database/sql honors ctx for cancellation/timeouts
        _, err := r.db.ExecContext(ctx, `INSERT INTO orders(id, total) VALUES($1,$2)`, o.ID, o.Total)
        return err
    }

Why it’s good: When the client disconnects or the 3s timeout hits, r.Context() is canceled → DB call bails out quickly.

You have three layers here:

    HTTP handler → handles incoming HTTP requests.

    Service layer → contains business logic.

    Repository layer → talks to the database.

The handler uses dependency injection:

    h is a *Handler struct.

    h.svc is a field on Handler that stores a reference to the service layer.

That service layer (*Service) has its own dependency: s.repo, the repository.


3️⃣ Breaking down the code flow

# Handler layer

    func (h *Handler) CreateOrder(w http.ResponseWriter, r *http.Request) {
        ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
        defer cancel()

        if err := h.svc.CreateOrder(ctx, parseOrder(r)); err != nil {
            http.Error(w, err.Error(), http.StatusGatewayTimeout)
            return
        }
        w.WriteHeader(http.StatusCreated)
    }

Purpose: Handle an HTTP request to create a new order.

    context.WithTimeout(r.Context(), 3*time.Second)

    Sets a deadline: all work done in this request must finish within 3 seconds, or it’s canceled.

    defer cancel()

    Cleans up resources when the request finishes (important to avoid leaks).

    parseOrder(r)

    Reads request body/form/JSON into an Order struct.

    h.svc.CreateOrder(...)

    Delegates the actual work to the service layer.

    If there’s an error, returns 504 Gateway Timeout (or whatever status you map).

    If successful, returns 201 Created.

# Service layer

    func (s *Service) CreateOrder(ctx context.Context, o Order) error {
        // do stuff, then persist
        return s.repo.Save(ctx, o)
    }

Purpose: Business logic for creating an order.

    Could include:

        Validation (o.Total > 0)

    Checking inventory

    Charging payment

    Calls s.repo.Save(ctx, o) to persist the order.

# Repository (Repo) layer

    func (r *Repo) Save(ctx context.Context, o Order) error {
        _, err := r.db.ExecContext(ctx, `INSERT INTO orders(id, total) VALUES($1,$2)`, o.ID, o.Total)
        return err
    }
Purpose: Interact with the database.

    Uses ExecContext instead of Exec so it respects context cancellation and timeouts.

    If the request times out or the client disconnects:

    The DB query is canceled early.

    No stuck queries or leaked connections.


# Pointer in Go

In Go web apps:

    Handlers usually have dependencies (services, configs) that don’t change often.

    These dependencies are stored in a struct.

    That struct is passed around as a pointer so all handlers share the same instance.

Example:


    handler := &Handler{svc: &Service{repo: &Repo{db: dbConn}}}
    http.HandleFunc("/orders", handler.CreateOrder)