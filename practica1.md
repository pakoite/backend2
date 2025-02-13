## chat.proto
```proto
syntax = "proto3";

package chat;

// Mensaje que representa un mensaje de chat
message ChatMessage {
  string user = 1;
  string message = 2;
}

// Servicio Chat que define la comunicación bidireccional de streaming
service ChatService {
  // El cliente y el servidor pueden enviar múltiples mensajes.
  rpc ChatStream(stream ChatMessage) returns (stream ChatMessage);
}
```

> protoc --go_out=. --go-grpc_out=. chat.proto


## server.go
```go
package main

import (
	"fmt"
	"io"
	"log"
	"net"

	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection"
	"your_project/chat" // Asegúrate de usar el path correcto donde generaste los archivos Go.
)

type server struct {
	chat.UnimplementedChatServiceServer
}

// Implementación del servicio de streaming
func (s *server) ChatStream(stream chat.ChatService_ChatStreamServer) error {
	for {
		// Recibir un mensaje del cliente
		message, err := stream.Recv()
		if err == io.EOF {
			// El cliente cerró la conexión
			return nil
		}
		if err != nil {
			log.Fatalf("Error al recibir el mensaje: %v", err)
		}

		// Mostrar el mensaje recibido
		fmt.Printf("%s dice: %s\n", message.User, message.Message)

		// Enviar una respuesta al cliente
		err = stream.Send(&chat.ChatMessage{
			User:    "Servidor",
			Message: "Recibido: " + message.Message,
		})
		if err != nil {
			log.Fatalf("Error al enviar la respuesta: %v", err)
		}
	}
}

func main() {
	// Crear el servidor gRPC
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("No se pudo escuchar en el puerto 50051: %v", err)
	}

	s := grpc.NewServer()
	chat.RegisterChatServiceServer(s, &server{})
	reflection.Register(s)

	// Iniciar el servidor
	fmt.Println("Servidor gRPC escuchando en el puerto 50051...")
	if err := s.Serve(lis); err != nil {
		log.Fatalf("Error al iniciar el servidor: %v", err)
	}
}
```

##client.go
```go
package main

import (
	"context"
	"fmt"
	"google.golang.org/grpc"
	"your_project/chat" // Asegúrate de usar el path correcto donde generaste los archivos Go.
	"log"
	"time"
)

func main() {
	// Conectar al servidor gRPC
	conn, err := grpc.Dial(":50051", grpc.WithInsecure(), grpc.WithBlock())
	if err != nil {
		log.Fatalf("No se pudo conectar al servidor: %v", err)
	}
	defer conn.Close()

	client := chat.NewChatServiceClient(conn)

	// Iniciar el streaming bidireccional
	stream, err := client.ChatStream(context.Background())
	if err != nil {
		log.Fatalf("Error al iniciar streaming: %v", err)
	}

	// Enviar un par de mensajes al servidor
	for i := 0; i < 5; i++ {
		// Enviar mensaje al servidor
		err := stream.Send(&chat.ChatMessage{
			User:    "Cliente",
			Message: fmt.Sprintf("Mensaje #%d", i+1),
		})
		if err != nil {
			log.Fatalf("Error al enviar mensaje: %v", err)
		}

		// Recibir respuesta del servidor
		resp, err := stream.Recv()
		if err != nil {
			log.Fatalf("Error al recibir respuesta: %v", err)
		}

		// Mostrar la respuesta
		fmt.Printf("Servidor: %s\n", resp.Message)
		time.Sleep(1 * time.Second)
	}
}
```
> go run server.go


> go run client.go
