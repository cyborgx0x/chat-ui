<template>
  <div class="row justify-center" style="padding: 16px">
    <div style="width: 100%">
      <div>
        {{ messages.map((message) => message.content).join("") }}
      </div>
      <div style="position: absolute; bottom: 20px">
        <input
          type="text"
          v-model="prompt"
          placeholder="Enter your prompt"
          @keydown="handleKeydown"
          style="height: 80px; width: 600px"
        />
        <button @click="sendPrompt" style="height: 80px; width: 100px">
          Send
        </button>
      </div>
    </div>
  </div>
</template>

<script>
function parseMessages(messages) {
  return messages.map((message) => message.content);
}
async function startStream(url, requestData, onData, onError, onDone) {
  try {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify(requestData),
    });

    const reader = response.body.getReader();

    return (async function read() {
      const { done, value } = await reader.read();

      if (done) {
        onDone();
        return;
      }

      try {
        const chunk = new TextDecoder().decode(value);
        onData(chunk);
      } catch (error) {
        onError(error);
      }

      // Read the next chunk
      read();
    })();
  } catch (error) {
    console.error("Error starting stream:", error);
    onError(error);
  }
}

export default {
  name: "ChatComponent",
  data() {
    return {
      messages: [],
      prompt: "",
    };
  },

  methods: {
    handleKeydown(event) {
      if (event.keyCode === 13) {
        this.sendPrompt();
      }
    },
    async sendPrompt() {
      const requestData = {
        model: "llama2", // Replace with your model name
        prompt: this.prompt,
      };

      startStream(
        "http://localhost:11434/api/generate",
        requestData,
        (chunk) => {
          console.log(chunk);
          const response = JSON.parse(chunk);
          this.messages.push({
            id: new Date().getTime(),
            content: response.response,
            sent: response.done,
          });
        },
        (error) => {
          // Handle error
          console.error("Streaming error:", error);
        },
        () => {
          // Handle stream completion
          console.log("Stream completed");
        }
      );

      this.prompt = ""; // Reset the prompt input after sending
    },
  },
};
</script>
