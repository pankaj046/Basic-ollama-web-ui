🧠 Ollama Setup + Basic Vue.js Web UI
=====================================

1\. Install Ollama (No Docker Required)
---------------------------------------

Run the following script to install Ollama on your system:

    curl -fsSL https://ollama.com/install.sh | sh

Once installed, start the Ollama server:

    ollama serve

2\. Pull a Model
----------------

Choose a model and pull it:

    ollama pull mistral

Other models: `phi3`, `llama3`

3\. Clone the Vue Web UI
------------------------

Clone the basic Ollama web UI built with Vue.js:

    git clone https://github.com/pankaj046/Basic-ollama-web-ui.git
    cd Basic-ollama-web-ui

4\. Install Dependencies
------------------------

Install project dependencies:

    npm install

5\. Update Ollama Port (If Needed)
----------------------------------

Ollama by default runs on `http://localhost:11434`.

If using a different port, update the API URL inside the Vue.js project (e.g., in `App.vue` or `main.js`):

    fetch('http://localhost:11434/api/generate', { ... })

6\. Run the Web Client
----------------------

Start the local development server:

    npm run dev

Then open your browser at:

    http://localhost:5173

7\. Chat with Ollama
--------------------

Your local Vue.js web client should now be connected to Ollama!

Type a message and start chatting with your local LLM.

🔗 Useful Links
---------------

*   [Ollama Model Library](https://ollama.com/library)
*   [GitHub: Basic Ollama Web UI](https://github.com/pankaj046/Basic-ollama-web-ui)