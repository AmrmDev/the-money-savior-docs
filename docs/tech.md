## Expense Registration

The user (myself) sends a message to a Telegram bot called **TheMoneySaviorBot**.

Example command:

"/gastei 17.84 uber credito"


The bot sends this message through an API to an AWS Lambda function (or via AWS API Gateway → AWS Lambda).

The Lambda function processes the received payload, validating whether the required fields are present, such as amount, description and method (command format: `/gastei amount description method`).

After validation, the Lambda function saves the expense (product, service, or investment) into a DynamoDB table, generating a unique expense ID and storing:
- expense ID
- date and time of the expense
- amount
- description
- payment method

Once the expense is successfully stored, the Lambda returns a `201 Created` response to the bot API.  
The bot then replies to the user with a confirmation message, such as:
- “Expense registered!”
- “Saved successfully!”

---

## Expense Query

The user sends a query command to the Telegram bot.

Example command:

"/consulta 15/12/2025"


The bot sends this request through an API to an AWS Lambda function (or via AWS API Gateway → AWS Lambda).

The Lambda function validates the request payload and then queries the DynamoDB table, filtering expenses by:
- specific day
- date range
- expense ID
- amount
- description
- payment method

The Lambda returns the formatted results to the bot via API, and the bot displays the data in a user-friendly format.

---

## Technologies Used

- AWS Free Tier
- Golang
- Insomnia
- Telegram Bot API

---

## Payload Contracts

As a user, I want to store the following data for each expense:
- amount
- description (e.g., Uber, grocery store, card game)
- expense date
- payment method (ticket, debit, credit)

No bank accounts or cards will be registered as separate entities.  
The payment method will be stored as a simple indexed field in the expense table to support filtering and reporting.

---

## Expense Registration Payload

**User → Bot command:**

"/gastei 21.74 uber credit"


**Bot → CreateExpenseLambda payload:**
```json
{
  "userId": "88d72777-834d-4264-83a1-7666b3a34fbf",
  "amount": 21.74,
  "description": "uber",
  "paymentMethod": "credit"
}
```

The Lambda function generates the expense ID and automatically assigns the current date and time before saving the record.

## Expense Query Payload

**User → Bot command:**

"/consulta 11/12/2025"

**Bot → QueryExpensesLambda payload:**

```json
{
  "userId": "88d72777-834d-4264-83a1-7666b3a34fbf",
  "date": "2025-12-11"
}
```

The Lambda function queries the DynamoDB table for expenses matching the provided date.