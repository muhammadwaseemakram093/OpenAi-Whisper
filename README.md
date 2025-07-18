# openai-whisper-talk

v0.0.2

**openai-whisper-talk** is a sample voice conversation application powered by **OpenAI** technologies such as [Whisper](https://platform.openai.com/docs/guides/speech-to-text), an automatic speech recognition (ASR) system, [Chat Completions](https://platform.openai.com/docs/guides/text-generation/chat-completions-api), an interface that simulates conversation with a model that plays the role of assistant, [Embeddings](https://platform.openai.com/docs/guides/embeddings/what-are-embeddings), converts text to vector data that can be used in tasks like semantic searching, and the latest [Text-to-Speech](https://platform.openai.com/docs/guides/text-to-speech), that turn text ito lifelike spoken audio. The application is built using [Nuxt](https://nuxt.com/docs/getting-started/introduction), a Javascript framework based on [Vue.js](https://vuejs.org/guide/introduction.html).

The application has two new features: [Schedule Management](#schedule-management) and [Long-Term Memory](#long-term-memory). With Schedule Management, you can command the chatbot to add, modify, delete, and retrieve scheduled events. The Long-Term Memory feature allows you to store snippets of information that the chatbot will remember for future reference. You can seamlessly integrate both functions [into your conversations](#sample-conversation---schedule-management) simply by interacting with the chatbot.

> **Update:** Updated the openai module to latest version and [replaced the embedding model](#embeddings-api) from `text-embedding-ada-002` to the [new v3 model `text-embedding-3-small`](https://openai.com/blog/new-embedding-models-and-api-updates).

---

**openai-whisper-talk**は、[Whisper](https://platform.openai.com/docs/guides/speech-to-text)（自動音声認識（ASR）システム）、[Chat completions](https://platform.openai.com/docs/guides/text-generation/chat-completions-api)（アシスタントの役割を果たすモデルとの会話をシミュレートするインターフェース）、[Embeddings](https://platform.openai.com/docs/guides/embeddings/what-are-embeddings)（セマンティック検索などのタスクで使用できるベクターデータにテキストを変換する）、そして最新の[Text-to-speech](https://platform.openai.com/docs/guides/text-to-speech)（テキストをリアルな話し言葉のオーディオに変える）など、OpenAI の技術を駆使したサンプル音声会話アプリケーションです。このアプリケーションは、[Vue.js](https://vuejs.org/guide/introduction.html)に基づいた Javascript フレームワークである[Nuxt](https://nuxt.com/docs/getting-started/introduction)を使用して構築されています。

このアプリケーションには、「[スケジュール管理](#schedule-management)」と「[永続メモリ](#long-term-memory)」の 2 つの新機能があります。スケジュール管理を使用すると、チャットボットにスケジュールイベントの追加、変更、削除、取得を指示できます。永続メモリ機能を使用すると、将来の参照のためにチャットボットが覚えておく情報のスニペットを保存できます。これらの機能を[チャットボットとの対話](#sample-conversation---long-term-memory)を通じてシームレスに統合することができます。将来的に、いくつかの機能強化、たとえばメールやメッセージング機能を追加することで、完全な個人アシスタントになるかもしれません。

# The App

![Main screen](./docs/app-main.png)

From the main page, you can choose which chatbot to engage with. Each chatbot has a distinct personality, speaks a different language, and possesses a unique voice. You can alter the name and personality of any chatbot by clicking the **Edit** button adjacent to its name. Currently, the user interface does not directly support the addition of new chatbots; however, you can manually add a chatbot and customize the voice and language settings for each by modifying the [assets/contacts.json](/assets/contacts.json) file.

![Edit Friend](./docs/friend-edit.png)

Additionally, you can personalize your profile by clicking on the avatar icon in the upper right corner. This allows you to enter your name and share details about yourself, enabling the chatbot to interact with you in a more personalized manner.

![Edit User](./docs/user-edit.png)

# Audio Capture

![Talk screen](./docs/app-talk.png)

Audio data is automatically recorded if sound is detected. A threshold setting is available to prevent background noise from triggering the audio capture. By default, this is set to **-45dB** (with -10dB representing the loudest sound cutoff). You can adjust this threshold by modifying the `MIN_DECIBELS`[^1] variable according to your needs.

When recording is enabled and no sound is detected for **3 seconds**, the audio data is uploaded and sent to the backend for transcription. It’s worth noting that in typical conversations, the average gap between each turn is around 2 seconds. Similarly, the pause between sentences when speaking is approximately the same. Therefore, I’ve chosen a duration that’s long enough to mean to wait for a reply. You can adjust this duration by editing the `MAX_COUNT`[^1] variable.

The system can continuously record audio data until a reply is received. Once the audio reply from the chatbot is received and played, audio recording is disabled to prevent inadvertent recording of the chatbot’s own response.

A text input is also provided if you want to write your messages.

[^1]: You can find these variables in [pages/talk/[id].vue](/pages/talk/[id].vue) file.

# Whisper

All recorded audio data is uploaded to the `public/upload` directory in WebM file format. Before submitting the audio file to the Whisper API, it is necessary to remove all silent segments. This step helps prevent the [well-known issue of hallucinations](https://github.com/openai/whisper/discussions/1369) generated by Whisper. For this same reason, it is recommended to set the `MIN_DECIBELS` value as high as possible, ensuring that only speech is recorded.

## Remove Silent Parts From Audio

To remove silent parts from the audio, we will be using `ffmpeg`. Be sure to [install it](#ffmpeg-setup).

```sh
ffmpeg -i sourceFile -af silenceremove=stop_periods=-1:stop_duration=1:stop_threshold=-50dB outputFile
```

In this command:

- `-1 sourceFile` specifies the input file.
- `-af silenceremove` applies the filter `silencerremove`.
- `stop_periods=-1` removes all periods of silence.
- `stop_duration=1` sets any period of silence longer than 1 second as silence.
- `stop_threshold=-50dB` sets any noise level below -50dB as silence.
- `outputFile` the output file.

To invoke this shell command in our api route, we will be using `exec` from `child-process` module.

After removing the silent parts, the file size should be checked. The final file size is usually much smaller than the original. During this check, files smaller than 16 KB are ignored, assuming that a file of this size, with a 16-bit depth, equates to roughly one second of audio. Anything shorter is likely inaudible.

```javascript
// simple file size formula
const audio_file_size = duration_in_seconds * bitrate;
```

The entire process, from eliminating silent parts to checking file size, is designed to ensure that only viable audio data is sent to the Whisper API. Our aim is to avoid hallucination and the unnecessary transmission of data.

Having confirmed the viability of our audio data, we then proceed to call the Whisper API.

```javascript
const transcription = await openai.audio.transcriptions.create({
  file: fs.createReadStream(filename),
  language: lang,
  response_format: "text",
  temperature: 0,
});
```

where `lang` is the [ISO 639-1](https://en.wikipedia.org/wiki/List_of_ISO_639-1_codes) code of the specified language of our chatbot. Please check the [list of the currently supported language](https://platform.openai.com/docs/guides/speech-to-text/supported-languages).

We opt for a simple `text` format since timestamps are not required, and we set the `temperature` parameter to zero to achieve deterministic output.

# Chat Completions API

After receiving the transcript from Whisper, we proceed to submit it to the Chat Completions API with function calling. We utilize the latest [OpenAI Node.js module (version 4)](https://www.npmjs.com/package/openai) that was released yesterday (2023/11/07), which includes an [updated function-calling format](https://platform.openai.com/docs/guides/function-calling). This newest iteration enables the invocation of multiple functions in a single request.

```javascript
let messages = [{ role: "system", content: system_prompt }];

let all_history = await mongoDb.getMessages();
if (all_history.length > 0) {
  const history_context = trim_array(all_history, 20);
  messages = messages.concat(history_context);
}

messages.push({ role: "user", content: user_query });

let response = await openai.chat.completions.create({
  temperature: 0.3,
  messages,
  tools: [
    { type: "function", function: add_calendar_entry },
    { type: "function", function: get_calendar_entry },
    { type: "function", function: edit_calendar_entry },
    { type: "function", function: delete_calendar_entry },
    { type: "function", function: save_new_memory },
    { type: "function", function: get_info_from_memory },
  ],
});
```

Let’s dissect the various components of this call in the subsequent sections.

## The System Prompt

The system prompt plays a crucial role in giving life to our chatbot. It is here where we establish its name and persona, based on the chatbot chosen by the user. We provide it with specific instructions on how to respond, along with a list of functions it can execute. We also inform it about the user's identity and some personal details. Finally, we set the current date and time, which is essential for activating calendar functions.

```javascript
const today = new Date();

let system_prompt =
  `In this session, we will simulate a voice conversation between two friends.\n\n` +
  `# Persona\n` +
  `You will act as ${selPerson.name}.\n` +
  `${selPerson.prompt}\n\n` +
  `Please ensure that your responses are consistent with this persona.\n\n` +
  `# Instructions\n` +
  `The user is talking to you over voice on their phone, and your response will be read out loud with realistic text-to-speech (TTS) technology.\n` +
  `Use natural, conversational language that are clear and easy to follow (short sentences, simple words).\n` +
  `Keep the conversation flowing.\n` +
  `Sometimes the user might just want to chat. Ask them relevant follow-up questions.\n\n` +
  `# Functions\n` +
  `You have the following functions that you can call depending on the situation.\n` +
  `add_calendar_entry, to add a new event.\n` +
  `get_calendar_entry, to get the event at a particular date.\n` +
  `edit_calendar_entry, to edit or update existing event.\n` +
  `delete_calendar_entry, to delete an existing event.\n` +
  `save_new_memory, to save new information to memory.\n` +
  `get_info_from_memory, to retrieve information from memory.\n\n` +
  `When you present the result from the function, only mention the relevant details for the user query.\n` +
  `Omit information that is redundant and not relevant to the query.\n` +
  `Always be concise and brief in your replies.\n\n` +
  `# User\n` +
  `As for me, in this simulation I am known as ${user_info.name}.\n` +
  `${user_info.prompt}\n\n` +
  `# Today\n` +
  `Today is ${today}.\n`;
```

## Context

All messages will be stored in **MongoDB**.

To manage tokens and avoid surpassing the model's maximum limit, we will only send the last 20 interactions. The `trim_array` function is designed to trim the message history if it surpasses 20 turns. This threshold can be adjusted to meet your specific requirements.

```javascript
// allow less than or equal to 15 turns
const history_context = trim_array(all_history, 15);
```

From the main screen, you have the option to erase the previous history for each chatbot.

![Clear messages](./docs/clear-messages.png)

## Function Calling

Please note that we have separated the handling of function calling ([function_call.js](/server/api/function_call.js)) from the main Chat Completions API call ([transcribe.php](/server/api/transcribe.js)). This distinction is made to address instances when text content is present while a function call is invoked, which is typically null. This separation enables the app to display the text while simultaneously processing the function call. I have also enclosed the process within a loop in the event that a second API call results in another function call. This could probably be handled more elegantly by implementing **streaming**, but I have yet to learn how to use streaming with Nuxt.

Our functions are categorized under two new features: **Schedule Management** and **Long-Term Memory**.

Let's first examine how we manage function calling with the new format. We must utilize the new **tools** parameter instead of the now-deprecated **functions** parameter to enable the invocation of multiple function calls.

```javascript
let response = await openai.chat.completions.create({
  temperature: 0.3,
  messages,
  tools: [
    { type: "function", function: add_calendar_entry },
    { type: "function", function: get_calendar_entry },
    { type: "function", function: edit_calendar_entry },
    { type: "function", function: delete_calendar_entry },
    { type: "function", function: save_new_memory },
    { type: "function", function: get_info_from_memory },
  ],
});
```

Check the **JSON Schema** of each functions from `lib/` directory.
Here is for `add_calendar_entry`:

```javascript
{
    "name": "add_calendar_entry",
    "description": "Adds a new entry to the calendar",
    "parameters": {
        "type": "object",
        "properties": {
            "event": {
                "type": "string",
                "description": "The name or title of the event"
            },
            "date": {
                "type": "string",
                "description": "The date of the event in 'YYYY-MM-DD' format"
            },
            "time": {
                "type": "string",
                "description": "The time of the event in 'HH:MM' format"
            },
            "additional_detail": {
                "type": "string",
                "description": "Any additional details or notes related to the event"
            }
        },
        "required": [ "event", "date", "time", "additional_detail" ]
    }
}
```

To dicuss how all these works, let's move on to the [next section](#schedule-management).

### Schedule Management

For **Schedule Management**, we have the following functions:

- [add_calendar_entry](/lib/add_calendar_entry.json), adds new calendar entry
- [edit_calendar_entry](/lib/edit_calendar_entry.json), edits calendar entry by event name
- [get_calendar_entry](/lib/get_calendar_entry.json), retrieves calendar entries by date
- [delete_calendar_entry](/lib/delete_calendar_entry.json), deletes calendar entry either by event name or by date[^2].

[^2]: If by date, item needs to be only one under the date otherwise an error will be returned.

All calendar entries will be stored in MongoDB. Please note that all entries will be accessible to all chatbots.

### Sample Conversation - Schedule Management

Let’s take a look at a sample chat conversation to demonstrate how these elements interact:

> user: good morning, jeeves. what's my schedule for today?

Function calling (invoke get_calendar_entry):

```javascript
{                                                                                                                                   role: 'assistant',
  content: null,
  tool_calls:[{
    id: 'call_4cCtmNlgYN5o4foVOQM9MIdA',
    type: 'function',
    function: { name: 'get_calendar_entry', arguments: '{"date":"2023-11-10"}' }
  }]
}
```

Function response:

```javascript
[
  {
    tool_call_id: "call_4cCtmNlgYN5o4foVOQM9MIdA",
    role: "tool",
    name: "get_calendar_entry",
    content:
      "{\n" +
      '  "message": "Found 3 entries",\n' +
      '  "items": [\n' +
      "    {\n" +
      '      "_id": "654b2805e51fcd815dea8e2d",\n' +
      '      "event": "Project presentation",\n' +
      '      "date": "2023-11-10",\n' +
      '      "time": "10:00",\n' +
      '      "additional_detail": "Important project presentation",\n' +
      '      "__v": 0\n' +
      "    },\n" +
      "    {\n" +
      '      "_id": "654d7b53ff0bf31de6c2af29",\n' +
      '      "event": "Meet Tony",\n' +
      '      "date": "2023-11-10",\n' +
      '      "time": "19:00",\n' +
      '      "additional_detail": "Electric Sheep Bar",\n' +
      '      "__v": 0\n' +
      "    },\n" +
      "    {\n" +
      '      "_id": "654d7d868751ea8704fc9f11",\n' +
      '      "event": "Dinner with Anna",\n' +
      '      "date": "2023-11-10",\n' +
      '      "time": "19:00",\n' +
      '      "additional_detail": "Electric Sheep Bar",\n' +
      '      "__v": 0\n' +
      "    }\n" +
      "  ]\n" +
      "}",
  },
];
```

Chat summary:

```javascript
{                                                                                                                                   role: 'assistant',
  content: 'Good morning, Jerry. Today, you have a project presentation at 10:00 AM and a dinner meeting with Anna at 7:00 PM at the Electric Sheep Bar. You also have a meeting with Tony at 7:00 PM at the same location.'
}
```

> user: it seems redundant to have separate entry for the same event. can you edit the event with Anna to include meeting Tony and delete the other event?

Function calling (invoked edit_calendar_entry and delete_calendar_entry):

```javascript
{                                                                                                                                   role: 'assistant',
  content: null,
  tool_calls: [
    {
    id: 'call_ZVAwO5YsXMuWr8876tyCIPIX',
    type: 'function',
    function: {
      name: 'edit_calendar_entry',
      arguments: '{"event": "Dinner Meeting with Anna", "date": "2023-11-10", "time": "19:00", "additional_detail": "Meeting with Tony included."}'
    }
  },
  {
    id: 'call_eMGuI8LxLbXFsLp8FduCrxgp',
    type: 'function',
    function: {
      name: 'delete_calendar_entry',
      arguments: '{"delete_type": "event_name", "event": "Meeting with Tony"}'
    }
  }
  ]
}
```

Function response:

```javascript
[
  {
    tool_call_id: "call_ZVAwO5YsXMuWr8876tyCIPIX",
    role: "tool",
    name: "edit_calendar_entry",
    content:
      '{\n  "message": "Entry edited",\n  "name": "Dinner Meeting with Anna"\n}',
  },
  {
    tool_call_id: "call_eMGuI8LxLbXFsLp8FduCrxgp",
    role: "tool",
    name: "delete_calendar_entry",
    content:
      '{\n  "name": "Meeting with Tony",\n  "message": "Entry deleted"\n}',
  },
];
```

Chat summary:

```javascript
{                                                                                                                                   role: 'assistant',
  content: 'The event "Dinner Meeting with Anna" has been updated to include the meeting with Tony, and the separate event "Meeting with Tony" has been deleted.'
}
```

> user: okay, just to be sure, can you tell me my schedule again for today?

Function calling (invoke get_calendar_entry):

```javascript
{                                                                                                                                   role: 'assistant',
  content: null,
  tool_calls: [
    {
    id: 'call_liqFn24Ds4qp0cy2V5NBg62v',
    type: 'function',
    function: { name: 'get_calendar_entry', arguments: '{"date":"2023-11-10"}' }
  }
  ]
}
```

Function response:

```javascript
[
  {
    tool_call_id: "call_liqFn24Ds4qp0cy2V5NBg62v",
    role: "tool",
    name: "get_calendar_entry",
    content:
      "{\n" +
      '  "message": "Found 2 entries",\n' +
      '  "items": [\n' +
      "    {\n" +
      '      "_id": "654b2805e51fcd815dea8e2d",\n' +
      '      "event": "Project presentation",\n' +
      '      "date": "2023-11-10",\n' +
      '      "time": "10:00",\n' +
      '      "additional_detail": "Important project presentation",\n' +
      '      "__v": 0\n' +
      "    },\n" +
      "    {\n" +
      '      "_id": "654d7d868751ea8704fc9f11",\n' +
      '      "event": "Dinner with Anna",\n' +
      '      "date": "2023-11-10",\n' +
      '      "time": "19:00",\n' +
      '      "additional_detail": "Meeting with Tony included.",\n' +
      '      "__v": 0\n' +
      "    }\n" +
      "  ]\n" +
      "}",
  },
];
```

Chat summary:

```javascript
{                                                                                                                                   role: 'assistant',
  content: 'Your schedule for today is as follows:\n' +
    '- Project presentation at 10:00 AM\n' +
    '- Dinner with Anna at 7:00 PM, which includes a meeting with Tony.'
}
```

> user: ah you forgot to add the location in the dinner. sorry but can you update it again?

Function calling (invoked edit_calendar_entry):

```javascript
{                                                                                                                                   role: 'assistant',
  content: null,
  tool_calls: [
    {
    id: 'call_eRTFMcz2l8qHO7nhp3msI3v8',
    type: 'function',
    function: {
      name: 'edit_calendar_entry',
      arguments: '{"event":"Dinner with Anna","date":"2023-11-10","time":"19:00","additional_detail":"Meeting with Tony at the Electric Sheep Bar."}'
    }
  }
  ]
}
```

Function response:

```javascript
{
    tool_call_id: 'call_eRTFMcz2l8qHO7nhp3msI3v8',
    role: 'tool',
    name: 'edit_calendar_entry',
    content: '{\n  "message": "Entry edited",\n  "name": "Dinner with Anna"\n}'
  }
```

Chat summary:

```javascript
{                                                                                                                                   role: 'assistant',
  content: 'The event "Dinner with Anna" has been updated to include the meeting with Tony at the Electric Sheep Bar.'
}
```

> user: okay, i want to confirm it again, can you tell me my schedule for today?

Function calling (invoked get_calendar_entry):

```javascript
{                                                                                                                                   role: 'assistant',
  content: null,
  tool_calls: [
    {
    id: 'call_HYlAj4BTeWcu2CxSp4zBYPuC',
    type: 'function',
    function: { name: 'get_calendar_entry', arguments: '{"date":"2023-11-10"}' }
  }
  ]
}
```

Function response:

```javascript
[
  {
    tool_call_id: "call_HYlAj4BTeWcu2CxSp4zBYPuC",
    role: "tool",
    name: "get_calendar_entry",
    content:
      "{\n" +
      '  "message": "Found 2 entries",\n' +
      '  "items": [\n' +
      "    {\n" +
      '      "_id": "654b2805e51fcd815dea8e2d",\n' +
      '      "event": "Project presentation",\n' +
      '      "date": "2023-11-10",\n' +
      '      "time": "10:00",\n' +
      '      "additional_detail": "Important project presentation",\n' +
      '      "__v": 0\n' +
      "    },\n" +
      "    {\n" +
      '      "_id": "654d7d868751ea8704fc9f11",\n' +
      '      "event": "Dinner with Anna",\n' +
      '      "date": "2023-11-10",\n' +
      '      "time": "19:00",\n' +
      '      "additional_detail": "Meeting with Tony at the Electric Sheep Bar.",\n' +
      '      "__v": 0\n' +
      "    }\n" +
      "  ]\n" +
      "}",
  },
];
```

Chat summary:

```javascript
{                                                                                                                                   role: 'assistant',
  content: 'Your schedule for today is as follows:\n' +
    '- Project presentation at 10:00 AM\n' +
    '- Dinner with Anna at 7:00 PM, which includes a meeting with Tony at the Electric Sheep Bar.'
}
```

### Long-Term Memory

For **Long-Term Memory**, we have the following functions

- [save_new_memory](/lib/save_new_memory.json), saves new memory.
- [get_info_from_memory](/lib/get_info_from_memory.json), retrieves info from memory based on search terms.

All previous functions have primarily focused on typical data retrieval and setting tasks. But for long term memory retrieval, we cannot just simply do keyword lookup. We need to consider the context of the query. We need to do semantic search. And in this regard, we will be using the **Embeddings API**. Let us proceed to the next section to discuss this further.

# Embeddings API

> **Update**: I have replaced `text-embedding-ada-002` with `text-embedding-3-small`. Based on my tests, the latter performs well enough. The cosine similarity scores between closely related answers and non-relevant answers are markedly distinct. Furthermore, the cost-performance of the v3 small model, [priced at $0.00002/1K tokens](https://openai.com/pricing), is a no brainer 😁. However, you need to convert your vector data from ada to v3 small since they are not compatible. Note that we also [lowered the threshold from 0.72 to 0.3](#cosine-similarity).

To put it simply, embeddings measures the relatedness of text strings. If we call the API, it will give us vector data of floating numbers associated with the input text.

```javascript
const embedding = await openai.embeddings.create({
  model: "text-embedding-3-small", //"text-embedding-ada-002",
  input: "The quick brown fox jumped over the lazy dog",
  encoding_format: "float",
});
```

To use this in our application, we will implement what they call **Retrieval-Augmented Generation** or simple RAG.

Initially, when new data is received from the save_new_memory function, we call the Embeddings API to generate its vector representation. This vector data is then stored in **MongoDB** for future use.

Subsequently, when a user submits a query that requires memory retrieval, the get_info_from_memory function is triggered.
We call the Embeddings API for the search parameters and compares them with the stored vector data using a simple cosine similarity. This comparison typically yields several matches with varying scores. We have set our threshold to a score of0.3 (0.72 for ada model) or higher, and we limit the results to a maximum of 10.

The results are then passed to the final Chat Completions API, which determines the most suitable response to the user’s query. The AI has the capability to select one or more results as the basis for its response, depending on the nature of the query. This is where the true power of AI is demonstrated. Rather than simply regurgitating all the information it receives, the AI analyzes the data and formulates an appropriate response. If the result from the RAG is deemed sufficient, it will generate a positive text response.

## Mongoose

To help facilitate the storage of vector data and other vector operations in **MongoDB**, I am using the [Mongoose](https://mongoosejs.com/) module. By far, this is the most painless solution to handle storage and retrieval of vector data.

First, we define our schema:

```javascript
const memoryVectorsSchema = new mongoose.Schema({
  chunks: [
    {
      embedding: [Number],
      text: String,
    },
  ],
});
```

Then we attach `getScore` function which executes **cosine similarity** between the document model and the embedded query string:

```javascript
memoryVectorsSchema.methods.getScore = function getScore(query_embedding) {
  return this.chunks.map((chunk) => {
    const dot_product = chunk.embedding.reduce(
      (sum, val, i) => sum + val * query_embedding[i],
      0
    );
    const magnitude = (vec) =>
      Math.sqrt(vec.reduce((sum, val) => sum + val * val, 0));
    const cosine_similarity =
      dot_product / (magnitude(chunk.embedding) * magnitude(query_embedding));
    return { text: chunk.text, score: cosine_similarity };
  });
};
```

Next, we compile our schema into a model:

```javascript
const MemoryVectors = mongoose.model("MemoryVectors", memoryVectorsSchema);
```

To save a vector data:

```javascript
const newVector = new MemoryVectors({
  chunks,
});

newVector.save();
```

### Cosine-Similarity

To search the stored vector data with the embeddings of the user query

```javascript
const cosineSimThreshold = 0.3; // Note: for text-embedding-ada-002, 0.72 but for v3 embed models, use 0.3
const maxResults = 10;
const maxFilesLength = 2000 * 3;

const vectors = await MemoryVectors.find();

const vector_result = vectors
  .map((v) => {
    return v.getScore(search_query);
  })
  .flat()
  .sort((a, b) => b.score - a.score)
  .filter((chunk) => chunk.score > cosineSimThreshold)
  .slice(0, maxResults);

const query_result =
  vector_result.length > 0
    ? vector_result
        .map((v) => v.text)
        .join("\n")
        .slice(0, maxFilesLength)
    : "";
```

That's it. Neat, huh?

### Sample Conversation - Long-Term Memory

To demonstrate how all these work, let's check a sample conversation.

> user: hey, tell me, what's your fave burger? cheese? bacon? mushroom?

> bot: Oh, for sure! I'm all about that bacon burger, you know? The crispy bacon just takes it to a whole new level. What about you?

> user: mushroom! with cheese! oh, i love it.

> bot: Totally! Mushroom and cheese are a match made in burger heaven. Have you tried making your own mushroom and cheese burger at home? It's so worth it!

> user: not yet. have you tried avocado?

> bot: Oh, totally! Avocado on a burger is like the cherry on top, you know? It adds that creamy, buttery goodness that takes the whole thing to the next level. Have you tried it?

> user: yeah. there is actually a funny story related to it. so i've had avocado burger before. then one time, me and my friends went to ski trip. along the way we stopped by a burger shop. while i was browsing the menu, i saw avocado and i pointed it when i ordered. do you know what happened next?

> bot: No way! What happened? Did they give you a burger with, like, a whole avocado on it? That would be hilarious!

> user: no. but close. they gave me a burger with slice of avocado. nothing else. it turns out, the one i picked on the menu is for vegan option!

> bot: Oh my gosh, that's too funny! So you ended up with a plain avocado burger instead of the one you were expecting. That's like a mix-up straight out of a comedy movie! Did you end up enjoying the avocado burger, though?

> user: i ate it alright but was not satisfied, of course. hey, can you save this funny memory, it will be good to recall sometimes

Function call (invoked save_new_memory):

```javascript
{                                                                                                                                   role: 'assistant',
  content: null,
  tool_calls: [
    {
    id: 'call_2TIYJOo9WJcprD7ozQsERoo4',
    type: 'function',
    function: {
      name: 'save_new_memory',
      arguments: '{"memory_title":"Avocado Burger Mix-Up","memory_detail":"Ordered an avocado burger at a ski trip burger shop, but got a plain avocado burger instead. Turns out, the one I picked on the menu was the vegan option! Hilarious mix-up.","memory_date":"2023-11-10","memory_tags":["funny","food","burger"]}'
    }
  }
  ]
}
```

At this point we call the embeddings API and get the vector data for the whole arguments and save it with the text to MongoDB.

Function response:

```javascript
[
  {
    tool_call_id: "call_2TIYJOo9WJcprD7ozQsERoo4",
    role: "tool",
    name: "save_new_memory",
    content: '{\n  "message": "New memory saved"\n}',
  },
];
```

> bot: Memory saved! Now you've got that funny mix-up story to look back on whenever you need a good laugh. It's like a little slice of life, you know?

Now, for retrieving info from memory

> user: by the way, do you know the burger joint that is closing this month? the one in shibuya? what was the name?

Function call (invoked get_calendar_entry):

```javascript
{                                                                                                                                   role: 'assistant',
  content: null,
  tool_calls: [
    {
    id: 'call_3PNs56LMrmDViZUnrxdBScGz',
    type: 'function',
    function: { name: 'get_calendar_entry', arguments: '{"date":"2023-11-30"}' }
  }
  ]
}
```

Function response:

```javascript
[
  {
    tool_call_id: "call_3PNs56LMrmDViZUnrxdBScGz",
    role: "tool",
    name: "get_calendar_entry",
    content: '{\n  "message": "No entries found"\n}',
  },
];
```

> bot: I couldn't find any info about a burger joint closing in Shibuya this month. Maybe it's just a rumor, you know? But if you find out the name, let me know!

> user: can you check your memory, i think i might have told you that before

Function call (invoked get_info_from_memory):

```javascript
{                                                                                                                                   role: 'assistant',
  content: null,
  tool_calls: [
    {
    id: 'call_WVBvjR0MKgaaTzik7QBwk3Ei',
    type: 'function',
    function: {
      name: 'get_info_from_memory',
      arguments: '{"search":["burger","Shibuya","closing"]}'
    }
  }
  ]
}
```

Here we get the embeddings for the **search** terms instead of the actual user query.
Then using the resulting vector data, we call `mongoDb.searchEntry` to search the stored vector data in MongoDB.

Function response:

```javascript
[
  {
    tool_call_id: "call_WVBvjR0MKgaaTzik7QBwk3Ei",
    role: "tool",
    name: "get_info_from_memory",
    content:
      "{\n" +
      '  "message": "Retrieved related info for burger,Shibuya,closing",\n' +
      `  "result": "score: 0.8513867910905064\\n title: Becker's ハンバーガーショップ閉店 detail: 千葉の人気なバーガーショップ「Becker's」が閉店するらしい。 tags: 閉店,ハンバーガー,千葉\\nscore: 0.8506170963709474\\n title: トリキバーガーの渋谷店閉店情報 detail: 渋谷のトリキバーガーの店舗が閉店することが決まりました。最後の日は今月の20日です。お店は渋谷の井の頭通りにあります。 date: 2023-11-10 tags: トリキバーガー,渋谷,閉店情報\\nscore: 0.8307398914702265\\n title: Becker's閉店情報 detail: 千葉の人気なバーガーショップBecker'sが閉店する。最後の日は今月の22日。 date: 2023-11-22 tags: 閉店情報,Becker's...\n` +
      "}",
  },
];
```

I am only showing partial result of the content but it contains several hits from the database that is related to the search terms. We send all the results to the Chat completions API and let the AI decide what to do.

> bot: Oh, right! The burger joint that's closing in Shibuya is the Triki Burger. Their Shibuya location is shutting down on the 20th of this month. It's sad to see them go, you know?

If you noticed, the partial result shown is in Japanese! So, definitely, this shows that we are not doing just simple keyword lookup here. You can add data of mixed language and query the memory using different language and chances are you will still get the answer. Let's try using our Russian bot, **_Vitaly_**. I will be using **Google Translate** since I don't speak Russian.

> user: Виталий, можешь вспомнить название моих любимых картофельных чипсов? (Vitaly, can you remember the name of my favorite potato chips?)

Function calling (invoked get_info_from_memory):

```javascript
{                                                                                                                                   role: 'assistant',
  content: null,
  tool_calls: [
    {
    id: 'call_8fXETNlDMuKEXsitUuqt5PlE',
    type: 'function',
    function: {
      name: 'get_info_from_memory',
      arguments: '{"search":["картофельные чипсы"]}'
    }
  }
  ]
}
```

So, again, we get the embeddings for the **search** term and use it against each vector data stored in MongoDB.

Function result:

```javascript
[
  {
    tool_call_id: "call_8fXETNlDMuKEXsitUuqt5PlE",
    role: "tool",
    name: "get_info_from_memory",
    content:
      "{\n" +
      '  "message": "Retrieved related info for картофельные чипсы",\n' +
      `  "result": "score: 0.7718838658905126\\n title: トムさんのお気に入りのパン detail: トムさんのお気に入りのパンはカレーパンとあんドーナツです。カレーパンはスパイシーなカレーとサクサクの生地が絶妙に合い、あんドーナツは甘さともちもち感が魅力です。\\nscore: 0.7560670627592779\\n title: Crisp Bacon Potato Chips detail: Brand: Dice tags: snack,potato chips,brand\\nscore: 0.7354783965092787\\n title: あまとうのあんみつ detail: 小樽のあまとうで食べるあんみつは絶品。...\n` +
      "}",
  },
];
```

You can see that the data result shown is mixed, with Japanese and English texts.

> bot: Твои любимые картофельные чипсы - Crisp Bacon Potato Chips. (Your favorite potato chips are Crisp Bacon Potato Chips.)

Okay, let's proceed to the final step.

# Text-to-Speech API

Upon receiving the text response from the preceding steps, we will now call the **Text-to-speech** API to enable our chatbot to vocalize the response

```javascript
try {
  const mp3 = await openai.audio.speech.create({
    model: "tts-1",
    voice: "alloy",
    input: "The quick brown fox jumps over the lazy dog.",
  });

  const buffer = Buffer.from(await mp3.arrayBuffer());
  await fs.promises.writeFile(filename, buffer);
} catch (error) {
  throw error;
}
```

## Voices

Since the [doc](https://platform.openai.com/docs/guides/text-to-speech) does not provide specific descriptions for each voice, so I asked **ChatGPT** to make some educated guesses based on their names:

1. **Alloy** (F): versatile and could be suitable for technical or instructional content, such as tutorials, educational material, or any content where clarity and precision are essential.

2. **Echo** (M): might be fitting for storytelling, audiobooks, or content that requires a more dramatic or narrative tone.

3. **Fable** (F): could be ideal for children's stories, fantasy content, or any narrative that requires a more playful or imaginative tone.

4. **Onyx** (M): might be suitable for reading serious literature, delivering news content, or any material that requires a more serious or formal delivery.

5. **Nova** (F): suitable for advertisements, motivational content, or any material that requires a positive and enthusiastic delivery.

6. **Shimmer** (F): suitable for lifestyle content, podcasts, or anything that requires an engaging, friendly, and inviting tone.

The generated audio data will then be saved to the `public/upload` directory in MP3 file format.
We will then send the link to the client side where a dynamic [HTMLAudioElement](https://developer.mozilla.org/en-US/docs/Web/API/HTMLAudioElement) will load and play it.

```javascript
const audioDomRef = new Audio();
audioDomRef.type = "audio/mpeg";
audioDomRef.addEventListener("loadedmetadata", handleAudioLoad);
audioDomRef.addEventListener("ended", handleAudioEnded);
audioDomRef.addEventListener("error", handleAudioError);
audioDomRef.src = audio_url;

function handleAudioLoad() {
  audioDomRef.value.play();
}
```

Please note that the recording of audio will be disabled while the audio response is playing. This ensures that the application does not record the bot’s response.

# FFMPEG Setup

[FFMPEG](https://ffmpeg.org/) is used to remove silent parts in the audio file.

To install ffmpeg command-line tool

```
# on Ubuntu or Debian
sudo apt update && sudo apt install ffmpeg

# on Arch Linux
sudo pacman -S ffmpeg

# on MacOS using Homebrew (https://brew.sh/)
brew install ffmpeg

# on Windows using Chocolatey (https://chocolatey.org/)
choco install ffmpeg

# on Windows using Scoop (https://scoop.sh/)
scoop install ffmpeg
```

# MongoDB Setup

[MongoDB](https://www.mongodb.com/try/download/community) will be used to store calendar entries and vector data for long-term memory function.

To install MongoDB Community Edition, please check this [page](https://www.mongodb.com/docs/manual/installation/).
You might also want to [install MongoDB Shell](https://www.mongodb.com/docs/mongodb-shell/install/) to view the database.

# App Setup

First, be sure that [ffmpeg](#ffmpeg-setup) and [MongoDB](#mongodb-setup) are installed in your system.

To clone the project repository and install the dependencies

```sh
$ git clone https://github.com/supershaneski/openai-whisper-talk.git myproject

$ cd myproject

$ npm install
```

Copy `.env.example` file and rename to `.env`, then open it and edit the items there with actual values. For the MongoDB items, you probably do not need to edit them unless you have a different setup.

```
NUXT_OPENAI_API_KEY=your-openai-api-key
NUXT_MONGODB_HOST_NAME=localhost
NUXT_MONGODB_PORT=27017
NUXT_MONGODB_DB_NAME=embeddingvectorsdb
```

Then to run the app

```sh
$ npm run dev
```

Open your browser to `http://localhost:5000/` (port number depends on availability) to load the application page.

## Using HTTPS

> **Note:** I have not yet tested this with the latest update

You might want to run this app using `https` protocol.
This is needed to enable audio capture using a separate device like a smartphone.

In order to do so, prepare the proper `certificate` and `key` files and edit `server.mjs` at the root directory.

Then buid the project

```sh
$ npm run build
```

Finally, run the app

```sh
$ node server.mjs
```

Now, open your browser to `https://localhost:3000/` (port number depends on availability) or use your local IP address to load the page.
