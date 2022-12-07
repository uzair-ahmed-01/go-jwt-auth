# Go JWT Auth (Fiber & Mongo)
For my Golang projects, I made a template to serve as a CookieCutter. I found that selecting Stack and creating the initial login and integrations, such as logger, database, etc., were the most difficult parts of starting a project. So I made the decision to design a template that was already functional.

The entire project is built on interfaces, allowing you to add your own logic and use it within the project.
For example, you can utilize many databases such as Postgres, MySQL, and others. Simply implement and use the interface.

## Stack
- Router: [Fiber 🚀](https://gofiber.io)
- Database: [Mongo 💾](https://www.mongodb.com/docs/drivers/go/current/) 
- Doc: [Swagger 📄](https://github.com/swaggo/swag)
- Logger: [Zap ⚡](https://github.com/uber-go/zap)
- Mocks: [gomock 💀](https://github.com/golang/mock)
- Deploy: [Docker 🐳](https://www.docker.com)
- CI: [Github Actions 🐙](https://docs.github.com/en/actions)

## Before the execution
- Modify the file `./config/env.json` with your parameters
- Install gomock `go install github.com/golang/mock/mockgen@v1.6.0`
- Install swag `go install github.com/swaggo/swag/cmd/swag@latest`

## Routes
You can also check in the route /swagger/index.html after run the project 🤩.

Note 📝: For add a private route you need to create it in the private router `v1Private`
inside the pkg/server/server.go file.

| Name          | Path             | Method | Description          | Request        | Response |
|---------------|------------------|--------|----------------------|----------------|----------|
| Register      | /api/v1/register | POST   | Create a new user    | email,password |          |
| Login         | /api/v1/login    | POST   | Login a user         | email,password | token    |
| Metrics       | /metrics         | GET    | Monitor for your API |                | html     |
| Documentation | /docs            | GET    | Documentation        |                | html     |

## How to use
In this example, let's say you want to create endpoints for blog posts.

1. Create a new folder inside the internal folder with the name of your entity. In this case `post`.
2. Create three tree folders inside the entity folder: 'application,' 'domain,' and 'infrastructure.'
3. Create the `ports` and `model` folders inside the domain folder.
4. Create a file with the name of your entity inside the model folder. `post.go` in this case, then create your struct `Post`.
    ```go
    package model
   
   import "time"  

    type Post struct {
        ID        string `json:"id" bson:"_id"`
        Title     string             `json:"title" bson:"title"`
        Content   string             `json:"content" bson:"content"`
        CreatedAt time.Time          `json:"created_at" bson:"created_at"`
        UpdatedAt time.Time          `json:"updated_at" bson:"updated_at"`
    }
    ```

5. Create the files `repository.go`, `handlers.go`, and `application.go` inside the ports folder.
6. The `repository.go`, `handlers.go`, and `application.go` files are where you define your interfaces.
   ```go
   package ports
   
   import "github.com/your_user/your_project/internal/post/domain/model"
   
   type PostRepository interface {
        Create(post *model.Post) error
        FindAll() ([]*model.Post, error)
        FindByID(id string) (*model.Post, error)
        Update(post *model.Post) error
        Delete(id string) error
    }
    ```
7. Modify the `scripts/generate-mocks.sh` file and add your three new interfaces.
   ```sh
   mockgen -destination=pkg/mocks/mock_post_application.go -package=mocks --build_flags=--mod=mod github.com/solrac97gr/go-jwt-auth/internal/post/domain/ports PostApplication &&
   mockgen -destination=pkg/mocks/mock_post_repository.go -package=mocks --build_flags=--mod=mod github.com/solrac97gr/go-jwt-auth/internal/post/domain/ports PostRepository &&
   mockgen -destination=pkg/mocks/mock_post_handlers.go -package=mocks --build_flags=--mod=mod github.com/solrac97gr/go-jwt-auth/internal/post/domain/ports PostHandlers
   ```
8. Run the `scripts/generate-mocks.sh` file.
9. It's time to put your interfaces into action. Create two folders called `repositories` and `handlers` inside the 'infrastructure' folder.
10. With the name of your interface implementation, create a file in the "repositories" folder. In this example, use `mongo.go` and implement the 'PostRepository' interface.
   ```go
    package repositories

    type MongoPostRepository struct {
        db *mongo.Database
        logger logger.LoggerApplication
        configurator config.ConfigApplication
    }
    
    func NewMongoPostRepository(db *mongo.Database) *MongoPostRepository {
        return &MongoPostRepository{db: db}
    }

    func (m *MongoPostRepository) Create(post *model.Post) error {
        _, err := m.db.Collection("posts").InsertOne(context.Background(), post)
        if err != nil {
			m.logger.Error("Error creating post", err)
            return err
        }
        return nil
    }
    .
    .
    .
   ```
11. With the name of your interface implementation, create a file in the "handlers" folder. In this case, use `http.go` and implement the 'PostHandler' interface.
   ```go
   package handlers

    type HTTPPostHandler struct {
        postApplication ports.PostApplication
        logger logger.LoggerApplication
        validator validator.ValidatorApplication
    }
    
    func NewHTTPPostHandler(postApplication ports.PostApplication) *HTTPPostHandler {
        return &HTTPPostHandler{postApplication: postApplication}
    }

    func (h *HTTPPostHandler) CreatePost(c *fiber.Ctx) error {
        post := &model.Post{}
        if err := c.BodyParser(post); err != nil {
            h.logger.Error("Error parsing post", err)
            return c.Status(http.StatusBadRequest).JSON(fiber.Map{"error": err.Error()})
        }
        if err := h.postApplication.Create(post); err != nil {
            h.logger.Error("Error creating post", err)
            return c.Status(http.StatusInternalServerError).JSON(fiber.Map{"error": err.Error()})
        }
        return c.Status(http.StatusCreated).JSON(fiber.Map{"message": "Post created successfully"})
    }
    .
    .
    .
   ```
12. Add your handlers to new routes depends on whether they are public or private. In this case, we'll add it to the private routes in the file `pkg/server/server.go.`
   ```go
   v1Private.Post("/posts", h.postHandler.CreatePost)
   ```
13. If you want to add it to the swagger documentation view, use a comment with a custom annotation for your handlers.
   ```go
   // @Summary Create a new post
   // @Description Create a new post
   // @Tags posts
   // @Accept json
   // @Produce json
   // @Param post body model.Post true "Post"
   // @Success 201 {object} fiber.Map
   // @Failure 400 {object} fiber.Map
   // @Failure 500 {object} fiber.Map
   // @Router /posts [post]
   func (h *HTTPPostHandler) CreatePost(c *fiber.Ctx) error {
        .
        .
        .
   }
   ```
14. Use the `/scripts/generate-docs.sh` command to create the swagger documentation.
15. Run the project using the script `scripts/run.sh` or the command `go run cmd/http/main.go`.

## Considerations
- The `scripts/generate-mocks.sh` file is used to create mock interfaces.
- The `scripts/generate-docs.sh` file is used to create the swagger documentation.
- The `scripts/run.sh` file is used to run the project.
- The `scripts/run-tests.sh` file is used to run the tests.
- The `scripts/run-tests-coverage.sh` file is used to run the tests with coverage.
- To avoid creating users with the same email address, make the mail field in the database unique (mongo index in this case).

## License
[Apache License 2.0]()
