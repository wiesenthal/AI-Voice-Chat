# AI-Voice-Chat
A Node.js web app using deepgram to transcribe voice to text sent to openai api, then stream the response back to be spoken one sentence at a time, optimizing for ultra-fast response times.
### Purpose
This software represents the foundation and starting point of my project to seamlessly integrate AI into daily life. It begins with voice-to-voice communication as a form more natural to human thought, and uses streaming to optimize for ultra-fast response time.
## Demo (Audio)
https://github.com/wiesenthal/AI-Voice-Chat/assets/26258920/1e75dda3-5bf1-4207-aa4c-f37a7dbc244a
## Architecture
![AI-Voice-Chat-Architecture](https://github.com/wiesenthal/AI-Voice-Chat/assets/26258920/c4b11906-99fb-4103-8bac-43905e227419)
## Structure Explanation
Two servers, named "orchestrator" and "brain", primarily compose the software. 
These servers divide the logic between 
1. the orchestration of serving, authenticating, and routing customer traffic and transforming input/output
2. the "brain" to process text and formulating prompts to generate and stream responses from AI.

The servers can be hosted locally or on AWS. I've currently turned off AWS to limit cost, but the architecture follows:
- The two Node.js servers are set up as Elastic Beanstalk EC2 instances within a VPC, along with an RDS MySQL database.
- An Internet Gateway allows traffic into the VPC to an elastic load balancer to sign HTTPS traffic for the orchestrator.
- Communications between the servers and to the database are securely within the VPC.
### ----- Orchestrator -----
The orchestrator controls the customer journey:
- Serves the frontend which by default uses Google Oauth2 to automatically log the user in. If it is able to securely retreive the user from session storage, it bypasses the Google OAuth2 step which can be mildly time consuming.
- Once obtained the user credentials, it serves the main "Talk To It" screen which allows the user to record audio and receive audio and text responses.

On the backend:
- The backend converts the user's audio to text using Deepgram's lightning fast transcription API, then sends this text to the brain server to be formulated into a prompt and streamed back. (Because of the long response time of LLM's like GPT-4, the backend uses streaming to drastically reduce response time.)
- As sentences are streamed back to the orchestrator from the brain, sentences are chunked out and queued to be read out by Amazon Polly, a speech-to-text service selected for its low cost and latency.
- The messages are read out per sentence because using smaller increments like words can make the spoken text sound choppy. Although the sentence chopping does have current limitations especially with numbers or code being read out. 

### ----- Brain -----
- The brain receives a message with the user's text, id, and commandID.
- It loads message history either from local session if available, or the database.
- It formulates this into a prompt compatible with the openAI chatCompletions API, and streams the response back to orchestrator.
- I implemented a local session storage in memory of message histories and users to limit database reads and writes.
- On disconnect or server kill, it saves the local session to database.

The brain is set up a seperate server to allow for different scaling than the orchestrator server. This allows for the future potential for the brain to open several threads per user for more advanced processing.

## Technologies
- React.js frontend
- Node.js with Express backend
- Google OAuth2 for user authentication
- Deepgram for transcription
- OpenAI GPT-3.5 and GPT-4, streaming
- Amazon Polly for speech-to-text
- MySQL Database

## Local Set Up
To set up the program locally perform the following:
- Clone this repository from github.
- You'll need a MySQL database with the following tables:
```sql
CREATE TABLE users (
    user_id VARCHAR(255) PRIMARY KEY,
    google_sub VARCHAR(255),
    name VARCHAR(255),
    email VARCHAR(255),
    payment_expiration_date BIGINT
);
```
```sql
CREATE TABLE messages (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id VARCHAR(255),
    role VARCHAR(255),
    content TEXT,
    command_id VARCHAR(255),
    timestamp TIMESTAMP BIGINT,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
```
- Generate an openAI API secret key at https://platform.openai.com/account/api-keys
- Follow https://developers.google.com/identity/gsi/web/guides/get-google-api-clientid to set up a google API client ID for OAuth2.
- Sign up for https://deepgram.com/ and get an API key.
- run `npm install` inside both folders "brain" and "orchestrator", and the frontend folder within "brain"
- .env files in each directory with the following format

inside brain:
```
OPENAI_API_KEY=<your secret key>
SESSION_SECRET_KEY=<an arbitrary string you can generate>

MODEL=<either gpt-3.5-turbo or gpt-4>

DB_NAME=<name of your database>
DB_USERNAME=<username for your database>
DB_ENDPOINT=<localhost, if hosting locally or the endpoint for your database>
DB_PASSWORD=<password for your database>
```
inside orchestrator:
```
DEEPGRAM_API_KEY=<secret key for https://deepgram.com/>
GOOGLE_CLIENT_ID=<google client ID for a google application to use google OAuth2>
SESSION_SECRET_KEY=<another arbitrary string you can generate>

BRAIN_RUNNING_LOCALLY=true
BRAIN_HOSTNAME=<unnecessary if hosting locally, but if cloud-hosting set the above to false and put the hostname of the brain server here>

DB_NAME=<name of your database>
DB_USERNAME=<username for your database>
DB_ENDPOINT=<localhost, if hosting locally or the endpoint for your database>
DB_PASSWORD=<password for your database>
```
- inside brain, `npm start`
- inside orchestrator, `cd frontend && npm run build && cd .. && npm start`, or use `./run` to do this as a macro.
- Navigate to localhost:1000 in your browser.
