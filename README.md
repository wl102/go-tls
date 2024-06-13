# go-tls
A best practice of tls with go 

## server
```go
package main

import (
	"crypto/tls"
	"log"
	"net/http"
	"strings"
)

func main() {
	cert, err := tls.LoadX509KeyPair("your certFile", "your keyFile")
	if err != nil {
		log.Fatalln(err)
	}
	config := &tls.Config{
		Certificates: []tls.Certificate{cert},
	}
    // use the security cipher suites 
	for _, v := range tls.CipherSuites() {
		config.CipherSuites = append(config.CipherSuites, v.ID)
	}

	srv := http.Server{
		Addr:      "localhost:8080",
		TLSConfig: config,
	}
	http.HandleFunc("/test", func(w http.ResponseWriter, r *http.Request) {
		srcIp := r.RemoteAddr
		addr := strings.Split(srcIp, ":")
		w.Write([]byte(addr[0]))
	})
	log.Fatalln(srv.ListenAndServeTLS("", ""))
}

```

## client
```go
package main

import (
	"crypto/tls"
	"crypto/x509"
	"fmt"
	"io"
	"log"
	"net/http"
	"os"
)

func main() {
	cas := x509.NewCertPool()
	f, err := os.Open("your certFile")
	if err != nil {
		log.Fatalf("open file err %v\n", err)
	}
	cert, err := io.ReadAll(f)
	if err != nil {
		log.Fatalf("read file err %v\n", err)
	}
	if ok := cas.AppendCertsFromPEM(cert); !ok {
		log.Fatalln("append cert from pem fail")
	}
	var config = &tls.Config{
        // don't use this setting in producing environment
		// InsecureSkipVerify: true,
		RootCAs: cas,
	}
    
	client := http.Client{
		Transport: &http.Transport{
			TLSClientConfig: config,
		},
	}
	res, err := client.Get("https://localhost:8080/test")
	if err != nil {
		log.Fatalln(err)
	}
	defer res.Body.Close()
	msg, _ := io.ReadAll(res.Body)
	fmt.Println(string(msg))
}
```