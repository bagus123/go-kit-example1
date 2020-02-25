#go-kit-example1

Let’s create a minimal Go kit service. For now, we will use a single main.go file for that.

###Your business logic
Your service starts with your business logic. In Go kit, we model a service as an interface.
```go
// StringService provides operations on strings.
import "context"

type StringService interface {
	Uppercase(string) (string, error)
	Count(string) int
}
```
That interface will have an implementation.
```go
import (
	"context"
	"errors"
	"strings"
)

type stringService struct{}

func (stringService) Uppercase(s string) (string, error) {
	if s == "" {
		return "", ErrEmpty
	}
	return strings.ToUpper(s), nil
}

func (stringService) Count(s string) int {
	return len(s)
}

// ErrEmpty is returned when input string is empty
var ErrEmpty = errors.New("Empty string")

```

###Requests and responses
In Go kit, the primary messaging pattern is RPC. So, every method in our interface will be modeled as a remote procedure call. For each method, we define request and response structs, capturing all of the input and output parameters respectively.
```go
type uppercaseRequest struct {
	S string `json:"s"`
}

type uppercaseResponse struct {
	V   string `json:"v"`
	Err string `json:"err,omitempty"` // errors don't JSON-marshal, so we use a string
}

type countRequest struct {
	S string `json:"s"`
}

type countResponse struct {
	V int `json:"v"`
}
```

###Endpoints
Go kit provides much of its functionality through an abstraction called an endpoint.

An endpoint is defined as follows (you don’t have to put it anywhere in the code, it is provided by go-kit):
```go
type Endpoint func(ctx context.Context, request interface{}) (response interface{}, err error)
```
It represents a single RPC. That is, a single method in our service interface. We’ll write simple adapters to convert each of our service’s methods into an endpoint. Each adapter takes a StringService, and returns an endpoint that corresponds to one of the methods.
```go
import (
	"context"
	"github.com/go-kit/kit/endpoint"
)

func makeUppercaseEndpoint(svc StringService) endpoint.Endpoint {
	return func(_ context.Context, request interface{}) (interface{}, error) {
		req := request.(uppercaseRequest)
		v, err := svc.Uppercase(req.S)
		if err != nil {
			return uppercaseResponse{v, err.Error()}, nil
		}
		return uppercaseResponse{v, ""}, nil
	}
}

func makeCountEndpoint(svc StringService) endpoint.Endpoint {
	return func(_ context.Context, request interface{}) (interface{}, error) {
		req := request.(countRequest)
		v := svc.Count(req.S)
		return countResponse{v}, nil
	}
}
```

###Transports
Now we need to expose your service to the outside world, so it can be called. Your organization probably already has opinions about how services should talk to each other. Maybe you use Thrift, or custom JSON over HTTP. Go kit supports many transports out of the box.

For this minimal example service, let’s use JSON over HTTP. Go kit provides a helper struct, in package transport/http.
```go
import (
	"context"
	"encoding/json"
	"log"
	"net/http"

	httptransport "github.com/go-kit/kit/transport/http"
)

func main() {
	svc := stringService{}

	uppercaseHandler := httptransport.NewServer(
		makeUppercaseEndpoint(svc),
		decodeUppercaseRequest,
		encodeResponse,
	)

	countHandler := httptransport.NewServer(
		makeCountEndpoint(svc),
		decodeCountRequest,
		encodeResponse,
	)

	http.Handle("/uppercase", uppercaseHandler)
	http.Handle("/count", countHandler)
	log.Fatal(http.ListenAndServe(":8080", nil))
}

func decodeUppercaseRequest(_ context.Context, r *http.Request) (interface{}, error) {
	var request uppercaseRequest
	if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
		return nil, err
	}
	return request, nil
}

func decodeCountRequest(_ context.Context, r *http.Request) (interface{}, error) {
	var request countRequest
	if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
		return nil, err
	}
	return request, nil
}

func encodeResponse(_ context.Context, w http.ResponseWriter, response interface{}) error {
	return json.NewEncoder(w).Encode(response)
}

```

### Installation

```shell
# 1. clone repository
git clone https://github.com/bagus123/go-kit-example1.git

# 2. downloads all dependencies and build binary
go build

# run from binary
./go-kit-example1

# or run from source
go run main.go

# note
# clean cache go
go clean --modcache

# remove unused module
go mod tidy
```

### Test
```shell
$ curl -XPOST -d'{"s":"hello, world"}' localhost:8080/uppercase
{"v":"HELLO, WORLD"}
$ curl -XPOST -d'{"s":"hello, world"}' localhost:8080/count
{"v":12}
```

