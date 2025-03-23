# go-message

[![Go Reference](https://pkg.go.dev/badge/github.com/emersion/go-message.svg)](https://pkg.go.dev/github.com/emersion/go-message)

A Go library for parsing and composing Internet messages, with a focus on email. It implements:

* [RFC 5322]: Internet Message Format
* [RFC 2045], [RFC 2046] and [RFC 2047]: Multipurpose Internet Mail Extensions (MIME)
* [RFC 2183]: Content-Disposition Header Field

## Overview

go-message provides a robust framework for working with Internet messages, particularly email. The library offers both high-level and low-level APIs for parsing, manipulating, and creating messages.

### Key Features

* **Streaming API**: Process messages efficiently without loading the entire content into memory
* **Automatic encoding and charset handling**: Seamlessly handle different character encodings
* **MIME support**: Full support for multipart messages and attachments
* **Email-specific functionality**: The `mail` subpackage provides specialized tools for email messages
* **DKIM-friendly**: Designed to work well with DKIM signatures
* **Low-level textproto package**: Access to the underlying wire format when needed

## Installation

```bash
go get github.com/emersion/go-message
```

To enable support for all character sets, also import:

```go
import (
    _ "github.com/emersion/go-message/charset"
)
```

## Core Concepts

### Entity

An `Entity` represents a message or a part of a multipart message. It has two main components:

* `Header`: Contains metadata about the entity
* `Body`: An `io.Reader` that provides access to the entity's content

### Header

A `Header` contains key-value pairs of metadata. The library provides convenient methods for working with common headers:

* `ContentType()`: Get the Content-Type and its parameters
* `SetContentType()`: Set the Content-Type and its parameters
* `ContentDisposition()`: Get the Content-Disposition and its parameters
* `SetContentDisposition()`: Set the Content-Disposition and its parameters
* `Text()`: Get a header field as plaintext (with charset decoding)
* `SetText()`: Set a header field as plaintext

### Multipart Messages

Multipart messages contain multiple parts, each with its own headers and body. The library provides:

* `MultipartReader`: An interface for iterating over parts in a multipart message
* `NewMultipart()`: Create a new multipart message from a list of entities

## Basic Usage

### Reading a Message

```go
import (
    "io"
    "log"
    
    "github.com/emersion/go-message"
    _ "github.com/emersion/go-message/charset" // Register all charsets
)

func readMessage(r io.Reader) {
    // Parse the message
    m, err := message.Read(r)
    if message.IsUnknownCharset(err) {
        // This error is not fatal
        log.Println("Warning: Unknown charset:", err)
    } else if err != nil {
        log.Fatal(err)
    }
    
    // Get the message content type
    contentType, params, _ := m.Header.ContentType()
    log.Println("Content-Type:", contentType)
    
    // Check if this is a multipart message
    if mr := m.MultipartReader(); mr != nil {
        // Process each part
        for {
            p, err := mr.NextPart()
            if err == io.EOF {
                break
            } else if err != nil {
                log.Fatal(err)
            }
            
            // Get the part's content type
            partType, _, _ := p.Header.ContentType()
            log.Println("Part Content-Type:", partType)
            
            // Read the part's body
            body, err := io.ReadAll(p.Body)
            if err != nil {
                log.Fatal(err)
            }
            log.Println("Part body:", string(body))
        }
    } else {
        // This is a single-part message
        body, err := io.ReadAll(m.Body)
        if err != nil {
            log.Fatal(err)
        }
        log.Println("Message body:", string(body))
    }
}
```

### Creating a Message

```go
import (
    "bytes"
    "io"
    "log"
    
    "github.com/emersion/go-message"
)

func createMessage() string {
    var b bytes.Buffer
    
    // Create a new message with headers
    var h message.Header
    h.SetContentType("multipart/alternative", nil)
    w, err := message.CreateWriter(&b, h)
    if err != nil {
        log.Fatal(err)
    }
    
    // Add an HTML part
    var htmlHeader message.Header
    htmlHeader.SetContentType("text/html", nil)
    htmlPart, err := w.CreatePart(htmlHeader)
    if err != nil {
        log.Fatal(err)
    }
    io.WriteString(htmlPart, "<h1>Hello World!</h1><p>This is an HTML part.</p>")
    htmlPart.Close()
    
    // Add a plain text part
    var textHeader message.Header
    textHeader.SetContentType("text/plain", nil)
    textPart, err := w.CreatePart(textHeader)
    if err != nil {
        log.Fatal(err)
    }
    io.WriteString(textPart, "Hello World!\n\nThis is a text part.")
    textPart.Close()
    
    // Finalize the message
    w.Close()
    
    return b.String()
}
```

## Working with Email Messages

The `mail` subpackage provides specialized functionality for email messages.

### Reading an Email

```go
import (
    "io"
    "log"
    "time"
    
    "github.com/emersion/go-message/mail"
    _ "github.com/emersion/go-message/charset" // Register all charsets
)

func readEmail(r io.Reader) {
    // Parse the email message
    mr, err := mail.CreateReader(r)
    if err != nil {
        log.Fatal(err)
    }
    
    // Print email metadata
    header := mr.Header
    
    if date, err := header.Date(); err == nil {
        log.Println("Date:", date)
    }
    
    if from, err := header.AddressList("From"); err == nil {
        log.Println("From:", from)
    }
    
    if to, err := header.AddressList("To"); err == nil {
        log.Println("To:", to)
    }
    
    if subject, err := header.Subject(); err == nil {
        log.Println("Subject:", subject)
    }
    
    // Process each part
    for {
        p, err := mr.NextPart()
        if err == io.EOF {
            break
        } else if err != nil {
            log.Fatal(err)
        }
        
        switch h := p.Header.(type) {
        case *mail.InlineHeader:
            // This is the message's text (can be plain text or HTML)
            contentType, _, _ := h.ContentType()
            log.Println("Got text part with content type:", contentType)
            
            body, err := io.ReadAll(p.Body)
            if err != nil {
                log.Fatal(err)
            }
            log.Println("Text:", string(body))
            
        case *mail.AttachmentHeader:
            // This is an attachment
            filename, _ := h.Filename()
            log.Println("Got attachment with filename:", filename)
            
            contentType, _, _ := h.ContentType()
            log.Println("Attachment content type:", contentType)
            
            // Read the attachment data (in a real application, you might want to
            // save this to a file instead of loading it into memory)
            data, err := io.ReadAll(p.Body)
            if err != nil {
                log.Fatal(err)
            }
            log.Printf("Attachment size: %d bytes", len(data))
        }
    }
}
```

### Creating an Email

```go
import (
    "bytes"
    "io"
    "log"
    "time"
    
    "github.com/emersion/go-message/mail"
)

func createEmail() string {
    var b bytes.Buffer
    
    // Create a new message with headers
    h := mail.NewHeader()
    h.SetDate(time.Now())
    h.SetAddressList("From", []*mail.Address{
        {Name: "John Doe", Address: "john@example.org"},
    })
    h.SetAddressList("To", []*mail.Address{
        {Name: "Jane Smith", Address: "jane@example.org"},
    })
    h.SetSubject("Hello, world!")
    
    // Create a multipart writer
    mw, err := mail.CreateWriter(&b, h)
    if err != nil {
        log.Fatal(err)
    }
    
    // Create a text part
    tw, err := mw.CreateInline()
    if err != nil {
        log.Fatal(err)
    }
    
    th := mail.NewInlineHeader()
    th.SetContentType("text/plain", nil)
    w, err := tw.CreatePart(th)
    if err != nil {
        log.Fatal(err)
    }
    io.WriteString(w, "Hi,\n\nThis is a test email.\n\nRegards,\nJohn")
    w.Close()
    tw.Close()
    
    // Create an attachment
    ah := mail.NewAttachmentHeader()
    ah.SetFilename("test.txt")
    ah.SetContentType("text/plain", nil)
    
    aw, err := mw.CreateAttachment(ah)
    if err != nil {
        log.Fatal(err)
    }
    io.WriteString(aw, "This is a test attachment.")
    aw.Close()
    
    // Finalize the message
    mw.Close()
    
    return b.String()
}
```

## Advanced Usage

### Transforming a Message

```go
import (
    "bytes"
    "io"
    "log"
    "strings"
    
    "github.com/emersion/go-message"
    _ "github.com/emersion/go-message/charset"
)

func transformMessage(r io.Reader) string {
    // Parse the original message
    m, err := message.Read(r)
    if err != nil {
        log.Fatal(err)
    }
    
    // Create a buffer for the transformed message
    var b bytes.Buffer
    
    // Create a writer with the same headers
    w, err := message.CreateWriter(&b, m.Header)
    if err != nil {
        log.Fatal(err)
    }
    
    // Define a recursive function to transform the message
    var transform func(w *message.Writer, e *message.Entity) error
    transform = func(w *message.Writer, e *message.Entity) error {
        if mr := e.MultipartReader(); mr != nil {
            // This is a multipart entity, transform each part
            for {
                p, err := mr.NextPart()
                if err == io.EOF {
                    break
                } else if err != nil {
                    return err
                }
                
                // Create a new part with the same headers
                pw, err := w.CreatePart(p.Header)
                if err != nil {
                    return err
                }
                
                // Transform this part recursively
                if err := transform(pw, p); err != nil {
                    return err
                }
                
                pw.Close()
            }
            return nil
        } else {
            // This is a leaf part, transform its content
            contentType, _, _ := e.Header.ContentType()
            
            // For text parts, add a footer
            if strings.HasPrefix(contentType, "text/") {
                body, err := io.ReadAll(e.Body)
                if err != nil {
                    return err
                }
                
                // Add a footer to the text
                body = append(body, []byte("\n\n-- \nTransformed by go-message")...)
                
                _, err = w.Write(body)
                return err
            } else {
                // For non-text parts, copy as-is
                _, err := io.Copy(w, e.Body)
                return err
            }
        }
    }
    
    // Apply the transformation
    if err := transform(w, m); err != nil {
        log.Fatal(err)
    }
    
    // Finalize the message
    w.Close()
    
    return b.String()
}
```

## API Reference

### Core Package (`message`)

#### Types

**Entity**
```go
type Entity struct {
    Header Header    // The entity's header
    Body   io.Reader // The decoded entity's body
}
```

**Header**
```go
type Header struct {
    textproto.Header
}
```

**Writer**
```go
type Writer struct {
    // contains filtered or unexported fields
}
```

**MultipartReader**
```go
type MultipartReader interface {
    io.Closer
    NextPart() (*Entity, error)
}
```

#### Functions

**Read**
```go
func Read(r io.Reader) (*Entity, error)
```

**ReadWithOptions**
```go
func ReadWithOptions(r io.Reader, opts *ReadOptions) (*Entity, error)
```

**New**
```go
func New(header Header, body io.Reader) (*Entity, error)
```

**NewMultipart**
```go
func NewMultipart(header Header, parts []*Entity) (*Entity, error)
```

**CreateWriter**
```go
func CreateWriter(w io.Writer, header Header) (*Writer, error)
```

### Mail Package (`mail`)

#### Types

**Header**
```go
type Header struct {
    message.Header
}
```

**Address**
```go
type Address struct {
    Name    string
    Address string
}
```

**Part**
```go
type Part struct {
    Header PartHeader
    Body   io.Reader
}
```

**Reader**
```go
type Reader struct {
    Header Header
    // contains filtered or unexported fields
}
```

**Writer**
```go
type Writer struct {
    // contains filtered or unexported fields
}
```

**InlineHeader**
```go
type InlineHeader struct {
    // contains filtered or unexported fields
}
```

**AttachmentHeader**
```go
type AttachmentHeader struct {
    // contains filtered or unexported fields
}
```

#### Functions

**CreateReader**
```go
func CreateReader(r io.Reader) (*Reader, error)
```

**CreateWriter**
```go
func CreateWriter(w io.Writer, h Header) (*Writer, error)
```

**NewReader**
```go
func NewReader(e *message.Entity) *Reader
```

**NewInlineHeader**
```go
func NewInlineHeader() InlineHeader
```

**NewAttachmentHeader**
```go
func NewAttachmentHeader() AttachmentHeader
```

## Character Set Support

By default, the library only supports UTF-8 encoding. To add support for additional character sets, import the charset package:

```go
import (
    _ "github.com/emersion/go-message/charset"
)
```

This adds support for most common character sets but increases binary size by about 1MB.

To register custom encodings:

```go
import (
    "github.com/emersion/go-message/charset"
    "golang.org/x/text/encoding/japanese"
)

func init() {
    charset.RegisterEncoding("iso-2022-jp", japanese.ISO2022JP)
}
```

## License

MIT

[RFC 5322]: https://tools.ietf.org/html/rfc5322
[RFC 2045]: https://tools.ietf.org/html/rfc2045
[RFC 2046]: https://tools.ietf.org/html/rfc2046
[RFC 2047]: https://tools.ietf.org/html/rfc2047
[RFC 2183]: https://tools.ietf.org/html/rfc2183
