# Dice Roll API

A small serverless HTTP API on AWS that rolls a die with a configurable number of sides. It is my first AWS Lambda deployment, built by following the AWS Free Tier onboarding tutorial *"Build a Serverless Dice Roll API"* (Esin Saribudak, AWS Builder Center, April 2026). The function code and template come from that tutorial. I did the setup, debugging and deployment on my own account.

**Status:** deployed and working in `eu-west-1` (Ireland), October 2026.

## What it does

`GET /Prod/roll?sides=N` returns a random roll from 1 to N as JSON.

```json
{ "sides": 20, "roll": 17 }
```

- If `sides` is missing, the API rolls a standard 6-sided die.
- If `sides` is outside 2 to 100, it returns HTTP 400 with `{"error": "Sides must be between 2 and 100."}`.

## Architecture

```
client ──HTTP GET──▶ Amazon API Gateway (REST API, /roll)
                          │ invokes
                          ▼
                     AWS Lambda (Node.js 24, 128 MB, 5 s timeout)
                          │ returns { statusCode, headers, body }
                          ▼
                     JSON response to client
```

- **AWS Lambda** runs the handler in `lambda/index.mjs` (ES module) only when a request arrives. No server to manage.
- **Amazon API Gateway** gives the function a public HTTPS endpoint and routes `GET /roll` to it.
- **AWS SAM (Serverless Application Model)**: `template.yaml` defines the function, the API event and the output URL as infrastructure as code. SAM turns it into an **AWS CloudFormation** stack, so the whole thing can be recreated or deleted with one command.
- **Amazon S3**: SAM uploads the packaged code to a managed S3 bucket that Lambda deploys from.
- **IAM**: SAM creates the execution role the function runs under.

## Project structure

```
dice-roll-api/
├── lambda/
│   └── index.mjs      # Lambda handler
├── template.yaml      # SAM / CloudFormation template
└── samconfig.toml     # saved deploy settings (stack name, region; no secrets)
```

## Deploying it yourself

Prerequisites: an AWS account, the AWS CLI and SAM CLI, and Node.js 24.

```bash
aws configure          # access key for an IAM user (not root), region e.g. eu-west-1
sam validate
sam build
sam deploy --guided    # first time; afterwards just `sam deploy`
```

The deploy prints the endpoint URL as the `DiceRollApi` output. Test it with:

```bash
curl "https://<api-id>.execute-api.<region>.amazonaws.com/Prod/roll?sides=20"
```

Tear everything down with `sam delete`.

## Problems I hit and how I fixed them

- **`AWS Region was not found` on `sam validate`.** The AWS CLI on my new Mac had no profile. I created an access key for my IAM user (CLI use case, not the root user), ran `aws configure` with region `eu-west-1`, and checked it with `aws sts get-caller-identity`.
- **`sam validate` with no template.** I had created the folders but not `template.yaml` yet. SAM needs the template in the project root, next to `lambda/`, not inside it.
- **Stack name rejected by CloudFormation.** I first entered `Mac Mini Sam`. Stack names must match `[a-zA-Z][-a-zA-Z0-9]*`, so spaces aren't allowed. Re-running `sam deploy --guided` with `dice-roll-api` fixed it, and the saved `samconfig.toml` was overwritten.
- **`package.json file not found` during `sam build`.** This is a warning, not an error. The function has no npm dependencies, so SAM just copies the source.

## What I learned

- The serverless request/response pattern: API Gateway passes query parameters in `event.queryStringParameters`, and a Lambda proxy integration must return `statusCode`, `headers` and a **stringified** `body`.
- Infrastructure as code with SAM and CloudFormation: one template makes the deployment repeatable and reversible.
- AWS account hygiene: work as an IAM user rather than root, use access keys only for the CLI, and choose the region on purpose.
- Cost model: Lambda and API Gateway charge per request, so an idle API costs nothing. This project runs within the AWS Free Tier.

## Known limitations

These are honest notes on the tutorial code as it stands, and the next things I'd change:

- **No authentication.** Anyone with the URL can call the API. API Gateway's default throttling is the only protection, so I don't publish the live URL here.
- **Loose input parsing.** `parseInt(params.sides) || 6` means `sides=abc` and `sides=0` silently fall back to 6 instead of returning 400, and `sides=2.5` is treated as 2.
- **No automated tests or CI yet.** I've only tested it by hand with `curl`.
- `Math.random()` is fine for games but not cryptographically secure.

## Next steps

- Stricter validation with unit tests (e.g. Node's built-in test runner) and a GitHub Actions workflow.
- `count` and `advantage` parameters (several dice; roll twice and keep the higher).
- Roll history in Amazon DynamoDB with a `GET /history` endpoint.

## Related

This is part of my AWS learning during the Higher Diploma in Software Development at Maynooth University, alongside my larger AWS data-engineering project (SECOM Semiconductor Pipeline).
