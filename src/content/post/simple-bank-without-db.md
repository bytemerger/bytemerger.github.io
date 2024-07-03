---
title: "Simple Bank API without a DB"
publishDate: "12 March 2022"
description: "A series for a small Spring Boot application for Banking without a mainstream DB"
tags: ["Java", "banking", "design", "spring"]
---

## Introduction

After a long time away from Java, I needed to write this small application. An interesting feature of this application quote-unquote is that it would be built without a mainstream DB, not even H2 database.
This restriction from using already existing DBs largely influenced my design decisions for the application. Here is the finished [code](https://github.com/bytemerger/digiFinance)

## Thoughts and Design...

I had about six endpoints to implement, which are listed below.

### Endpoints

- `GET \account_info\:accountNumber`
- `GET \account_statement\:accountNumber`
- `POST \deposit`
- `POST \withdrawal`
- `POST \create_account`
- `POST \login`

With the endpoints, I just decided to have at first two controllers `AccountController`,`TransactionController`, and two services that would serve the two controllers above.

Generally, it makes sense to start applications designs from the models and data relationships. However, in my case, I had a challenge. I did not know how to persist data or even make my system have some kind of a past life (memory) which is where the design becomes more interesting.

First, I thought 🧐 maybe I could store my data somehow in memory 😄😄😄 but every request is different. From my experience with scripting languages, the application context exits only between request and response. A more reasonable solution is to keep the data in a singleton class that is managed by the spring container with `@ApplicationScope`. I guess every request would have the same object 🤔 - java is compiled.

Would like feedback on this if you have tried it…🙃

Well, I think a more robust solution would be to store a small DB of my own in a file. This is the basics of every database - just that the data structure is more efficient and well thought out. I ended up just storing my data in a JSON file; you would have guessed already.
The decision to have this JSON DB would introduce a new service `StoreService`. This service is for accessing, mapping, and writing to the file.

To achieve reusability and readability, I decided to take care of the errors in the general handler class using `@ControllerAdvice`. Welcome to the theatre of magic - Spring Annotations.

Although the magic nature of spring is outrageous, one aspect I particularly appreciate is the request validation process. This application had constraints in some requests. For instance, you should not deposit an amount greater than 1000000.00 or less than 1.00. The ease of just defining a new Data Transfer Object(DTO) with validation annotations `@NotNull, @NotBlank, etc` and using `@Valid` on the request body in the controller is something I like. However, I wonder what the DTO package would look like with many requests.

Every other decision at this point is generic with most spring boot applications.

## Data

The next step is to define how the data would look in the JSON file. I decided to use a Map of Maps to allow for O(1) read instead of looping through arrays to find accounts or transactions.

so the `Map<String, Map<String, Object>>` map of maps which would look like

```json
{
	"accounts": {
		"6513138112": {
			"accountName": "New User",
			"accountNumber": "6513138112",
			"balance": 800.0,
			"password": "$2a$10$Jqh6rXBNfVKccGD7oA1Rl.PqiUdhcYND3Z7L6D5L5TTqLj9B5SEvy"
		}
	},
	"transactions": {
		"6513138112": {
			"list": [
				{
					"transactionDate": "2022-02-26T22:08:08.063376",
					"transactionType": "Deposit",
					"narration": "",
					"amount": 900.0,
					"balanceAfterTran": 1700.0
				},
				{
					"transactionDate": "2022-02-26T22:16:39.848951",
					"transactionType": "Deposit",
					"narration": "",
					"amount": 100.0,
					"balanceAfterTran": 1800.0
				}
			]
		}
	}
}
```

I used Jackson for mapping from JSON to `Map<String, Map<String, Object>>` where `Object` would either be `Account` or `TranactionList`.

`TransactionList` is just a data class for lists of `Transaction` Objects see implementation below

```java
package com.bank.demo.data;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import lombok.Data;

import java.util.List;

@Data
@JsonIgnoreProperties(ignoreUnknown = true)
public class TransactionList {
    private List<Transaction> list;

    public TransactionList(List<Transaction> list) {
        this.list = list;
    }
    public void addToList(Transaction transaction) {
        this.list.add(transaction);
    }

    public TransactionList(){
        super();
    }
}
```

You can check out the rest of the implementations in the [code](https://github.com/bytemerger/digiFinance) on GitHub. You can also contribute or improve the code. I would appreciate your feedback…
In the next part, I would write about securing and completing the application stay tuned.
