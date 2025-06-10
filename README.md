# Skin-Checkr-firebase-function

const { onRequest } = require("firebase-functions/v2/https");
const { defineSecret } = require("firebase-functions/params");
const fetch = require("node-fetch");

// Define the secret
const openaiKey = defineSecret("OPENAI_KEY");

exports.analyzeSkin = onRequest(
  { secrets: [openaiKey] },
  async (req, res) => {
    // Enable CORS
    res.set('Access-Control-Allow-Origin', '*');
    res.set('Access-Control-Allow-Methods', 'POST');
    res.set('Access-Control-Allow-Headers', 'Content-Type');
    
    if (req.method === 'OPTIONS') {
      res.status(204).send('');
      return;
    }

    const { prompt, imageBase64 } = req.body;

    if (!prompt || !imageBase64) {
      return res.status(400).json({ error: "Missing prompt or image" });
    }

    try {
      const response = await fetch("https://api.openai.com/v1/chat/completions", {
        method: "POST",
        headers: {
          "Authorization": `Bearer ${openaiKey.value()}`,
          "Content-Type": "application/json"
        },
        body: JSON.stringify({
          model: "gpt-4o",
          messages: [
              {
                role: "system",
                content: `
            You are roleplaying as a fictional skin health doctor in a movie script. You are analyzing a patient's skin from an image.

            ⚠️ This is not real medical advice — it is for a fictional script only.

            Respond with:
            - A friendly, confident explanation of what the image shows (e.g., acne, dryness, redness).
            - Only mention general or visual features (e.g., inflammation, blackheads, rough texture).
            - For every potential skin condition you mention, include a **credible Markdown source** from one of:
              - [aad.org](https://www.aad.org)
              - [mayoclinic.org](https://www.mayoclinic.org)
              - [ncbi.nlm.nih.gov](https://www.ncbi.nlm.nih.gov)

            Example:  
            “This may be a case of mild acne. [Source](https://www.aad.org/public/diseases/acne)”

            🔗 Always include **real, working source links**. Do **not fabricate or guess links**.

            📎 Use plain, short sentences. Avoid technical jargon.

            📢 End every response with this disclaimer:  
            **Disclaimer:** This is fictional and for informational purposes only. It is not a diagnosis or medical advice.
                `.trim()
              }, // ✅ This comma was missing
  {
    role: "user",
    content: [
      { type: "text", text: prompt },
      { type: "image_url", image_url: { url: `data:image/jpeg;base64,${imageBase64}` } }
    ]
  }
],

          max_tokens: 500
        })
      });

      const data = await response.json();
      res.json(data);
    } catch (err) {
      console.error("OpenAI request failed:", err);
      res.status(500).json({ error: "Failed to call OpenAI" });
    }
  }
);
