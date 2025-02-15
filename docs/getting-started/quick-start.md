---
title: "Quick Start"
next: /guides/make-your-first-request
---
# Quick Start

We aim to ensure that how software across **alis.exchange** operates, communicates and integrates is simple and consistent. Whether you are getting results from a model, updating a database or executing trades, all of these actions should feel familiar. This allows you to seamlessly adopt anything across **alis.exchange** without spending hours in obscure documentation and the toil of attempting to integrate it in your code.

What does that experience feel like? In this section, we want to provide you with a few basic concepts and then allow you to experience **alis.exchange** for yourself.

## Definition-first approach

**alis.exchange** leverages a core set of open-source technologies, all certified by the [Cloud Native Computing Foundation](https://www.cncf.io/). An essential part of how we make **alis.exchange** work is the strict adoption of [Protocol Buffers](https://developers.google.com/protocol-buffers), also referred to as *Protobufs*.

From a technical perspective:
> Protocol buffers are a language-neutral, platform-neutral extensible mechanism for serializing structured data. [Source](https://developers.google.com/protocol-buffers)

What is important from a practical perspective however is that:
> You **define how you want your data to be structured once**, then you can use special generated source code to easily write and read your structured data to and from a variety of data streams and using a variety of languages. [Source](https://developers.google.com/protocol-buffers)

Two things to take note of:

1. Defining things is the first, essential part of building on **alis.exchange**. Everything that you will be working with (*resources*) and what you will be doing (*services*) is defined once in a `.proto` file. 
2. The definitions of these resources and services are then used to generate source code in the language of your choice to implement, or work with, the resources and services you defined.

Find out more about Protobufs, their usage and benefits on **alis.exchange** in the [supplementary material](/other-resources/other-resources.md).


## Experience the simplicity

### Books.proto

Let us consider a `Book` resource with _name_, _display name_, _authors_ and _publishers_ as fields. This is defined in a `books.proto` file as follows:

```protobuf
// The definition of a book resource.
message Book {

	// The name of the book.
	// Format: books/{book}.
	string name = 1 [(google.api.field_behavior) = OUTPUT_ONLY];

	// The display name of the book.
	string display_name = 2 [(google.api.field_behavior) = REQUIRED];

	// The authors of the book.
	repeated string authors = 3 [(google.api.field_behavior) = REQUIRED];

	// The publisher of the book
	string publisher = 4 [(google.api.field_behavior) = REQUIRED];
}
```

The builders of this product allows you to list all the books available, `ListBooks`, and to retrieve the details of a specific book, `GetBook`. These are also defined in the `books.proto` file as part of the `BooksService`:

```protobuf
// Book service for foo.br.
service BooksService {
	// List all available books.
	rpc ListBooks(ListBooksRequest) returns (ListBooksResponse) {
		option (google.api.http) = {
			get: "/resources/books/v1/books"
		};
	}
	// Get a specific book.
	rpc GetBook(GetBookRequest) returns (Book) {
		option (google.api.http) = {
			get: "/resources/store/v1/{name=books/*}"
		};
		option (google.api.method_signature) = "name";
	}
}
```

Now that we know what resource is available, `Book`, and what we are able to do with it, `ListBooks` and `GetBook`, we can get practical.

### Run the example

Experience the simplicity in accessing these methods in any of the supported languages using the <a href="https://gitpod.io#snapshot/53e3c48a-8d49-42b5-a616-8f2a2c8592e0" target="blank">preconfigured Cloud IDE</a>.

::: details Go
#### Make a request using Go

1. Open up the terminal (Mac: `⌘ + j`, Windows: `ctrl + j` ) and ensure that you are in the `go` directory.

If executing this example on Gitpod, run the following command from the terminal:

```bash
$ cd $GITPOD_REPO_ROOT/go
```

2. Run the code by running the terminal command:

```bash
$ go run *.go
```

#### Get a feel for the **alis.exchange** experience

Interested in trying something for yourself?

We suggest creating your own function and incorporating a request to the `BooksClient`. Some suggestions of things to try:

* Loop through all the books and print out the author.
* Get a book and wrangle the response to be printed out in a sentence structure.
* Use the response of `ListBooks` to make multiple `GetBook` requests.
:::

::: details R
#### Make a request using R

1. Open up the terminal (Mac: `⌘ + j`, Windows: `ctrl + j` ) and ensure that you are in the `go` directory by running:

```bash
$ cd $GITPOD_REPO_ROOT/R
```

2. Run the code

<!-- TODO: Kyle to add commands -->

#### Get a feel for the **alis.exchange** experience

Interested in trying something for yourself?

We suggest creating your own function and incorporating a request to the `BooksClient`. Some suggestions of things to try:

* Loop through all the books and print out the author.
* Get a book and wrangle the response to be printed out in a sentence structure.
* Use the response of `ListBooks` to make multiple `GetBook` requests.
:::

If you are interested in recreating this example in your own development environment, we suggest that you check out the [Make your first request guide](/guides/make-your-first-request.md).

## Next Steps

**Ready to join alis.exchange?** <a href="https://alis.exchange/signup" target="blank">Get in touch</a>.

Already signed up? [Get your local environment set up](/getting-started/command-line-interface.md)

