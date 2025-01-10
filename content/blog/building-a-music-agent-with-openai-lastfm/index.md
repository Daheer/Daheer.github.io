# Building a Music Recommendation Agent with OpenAI and Last.fm API
In this blog post, we'll walk through the process of building a **music recommendation agent** using **OpenAI's GPT-4 model** and the **Last.fm API**. We'll build a cool agent which will be able to identify a song and its artist from user-provided lyrics and then recommend similar tracks based on that information. By the end of this post, you'll have a fully functional agent that can assist users in discovering new music.

## What Exactly is an Agent?
An **agent** in the context of AI is a system that can perceive its environment, make decisions, and take actions to achieve specific goals. In this project, our agent will:

1. **Perceive**: Take user-provided lyrics as input.

2. **Decide**: Use OpenAI's GPT-4 model to identify the song title and artist.

3. **Act**: Call the Last.fm API to fetch similar tracks and display them to the user.

## Prerequisites
Before we dive into the code, make sure you have the following:

- **OpenAI API Key**: You can get this from the OpenAI website.

- **Last.fm API Key**: Sign up on Last.fm to get your API key.

## Install Dependencies
First, we need to install the necessary libraries and import our API keys. We'll use the `openai` library to interact with the GPT-4 model and the `requests` library to make API calls to Last.fm.

```bash
# Install the OpenAI library
pip install openai requests
```

## Setup Environment
```python
# Import Google Colab's userdata to securely access API keys
from google.colab import userdata

# Fetch API keys from userdata
OPENAI_API_KEY = XXX
LAST_FM_API_KEY = XXX

# Import necessary libraries
import json
from openai import OpenAI
import requests

# Specify the GPT model to use
GPT_MODEL = "gpt-4o"

# Initialize the OpenAI client
client = OpenAI(api_key=OPENAI_API_KEY)
```

## Defining the Tools
To enable the GPT model to call external functions (like fetching recommendations from Last.fm), we need to define a **tool**. A tool is essentially a function that the model can invoke during its reasoning process. In this case, we'll define a tool called `get_recommendations` that takes a song's title and artist as input and returns a list of similar tracks.

Here’s how we define the tool:

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_recommendations",
            "description": "Get a list of recommendations based on a song's title and artist.",
            "parameters": {
                "type": "object",
                "properties": {
                    "track": {
                        "type": "string",
                        "description": "The name of the track to get recommendations for."
                    },
                    "artist": {
                        "type": "string",
                        "description": "The name of the artist to get recommendations for."
                    }
                },
                "required": ["track", "artist"]
            }
        }
    }
]
```

### Breakdown

`type`: Specifies that this is a function tool.

`name`: The name of the function (`get_recommendations`).

`description`: A brief description of what the function does.

`parameters`: Defines the input parameters for the function. In this case, the function requires a `track` (song title) and an `artist`.

`required`: Specifies that both `track` and `artist` are mandatory parameters.

---

## Implementing the Recommendation Function
Next, we'll implement the `get_recommendations` function. This function will make a GET request to the Last.fm API and parse the response to get similar tracks.

```python
def get_recommendations(track: str, artist: str):
    """Function to get a list of recommendations based on a song's title and artist."""
    try:
        # Define the Last.fm API endpoint
        url = "https://ws.audioscrobbler.com/2.0/"
        
        # Set up the query parameters
        params = {
          "method": "track.getsimilar",
          "artist": artist,
          "track": track,
          "api_key": LAST_FM_API_KEY,
          "format": "json",
          "limit": 10,  # Limit the number of recommendations to 10
          "autocorrect": 1,  # Enable autocorrect for track and artist names
        }
        
        # Make the GET request to the Last.fm API
        response = requests.get(url, params=params)
        
        # Check if the request was successful
        if response.status_code == 200:
            # Parse the JSON response
            similar_tracks = response.json().get('similartracks', {}).get('track', [])
            
            # Extract track names and artists from the response
            tracks = [track['name'] for track in similar_tracks]
            artists = [track['artist']['name'] for track in similar_tracks]
            
            # Format the results into a readable string
            results = f"Similar Tracks:\n" + "\n".join([f"{track} by {artist}" for track, artist in zip(tracks, artists)])
        else:
            # Handle errors if the request fails
            print("Failed to get recommendations", response.status_code, response.text)
    except Exception as e:
        # Handle any exceptions that occur during the process
        results = f"Failed with error: {e}"
    
    return results
```

- The function takes two inputs: `track` (song title) and `artist`.

- It constructs a URL and query parameters for the Last.fm API.

- It makes a GET request to the API and checks if the response is successful, then extracts the list of similar tracks and formats them into a readable string. If the request fails, it prints an error message.

## Creating the Music Recommendation Agent
Let's put everything together to create the **music recommendation agent**. The plan:

1. Take user-provided lyrics.

2. Use the GPT-4 model to identify the song title and artist.

3. Call the `get_recommendations` function to fetch similar tracks.

4. Display the recommendations to the user.

```python
def Question3(lyrics: str):
    # Initialize the messages list for the GPT model
    messages = []
    
    # Add a system message to define the agent's role
    messages.append({"role": "system", "content": """
                                        You are a helpful music recommendation agent.
                                        You are able to detect the track and artist of a song from user-provided lyrics
                                        and recommend a list of songs based on the track and artist.
                                        """})
    
    # Add the user-provided lyrics to the messages list
    messages.append({"role": "user", "content": f"Lyrics {lyrics}"})

    # Call the GPT-4 model with the messages and tools
    response = client.chat.completions.create(
        model=GPT_MODEL,
        messages=messages,
        tools=tools,
        tool_choice={"type": "function", "function": {"name": "get_recommendations"}}
    )

    # Extract the response message from the model
    response_message = response.choices[0].message
    messages.append(response_message)

    # Check if the model wants to call a tool
    tool_calls = response_message.tool_calls
    if tool_calls:
        # Extract details of the tool call
        tool_call_id = tool_calls[0].id
        tool_function_name = tool_calls[0].function.name
        track = json.loads(tool_calls[0].function.arguments)['track']
        artist = json.loads(tool_calls[0].function.arguments)['artist']

        # If the tool call is for `get_recommendations`, execute the function
        if tool_function_name == 'get_recommendations':
            print("============================================")
            print(f"Calling function {tool_function_name}")
            print(f"Track: {track}")
            print(f"Artist: {artist}")
            
            # Call the `get_recommendations` function
            results = get_recommendations(track=track, artist=artist)
            
            # Print the results
            print("============================================")
            print(results)
            print("============================================", "\n\n")

            # Add the results back to the messages list
            messages.append({
                "role": "tool",
                "tool_call_id": tool_call_id,
                "name": tool_function_name,
                "content": str(results)
            })

            # Send the results back to the GPT model for a final response
            model_response_with_function_call = client.chat.completions.create(
                model=GPT_MODEL,
                messages=messages,
            )
            
            # Print the final response from the model
            print(model_response_with_function_call.choices[0].message.content)
        else:
            # Handle errors if the function doesn't exist
            print(f"Error: function {tool_function_name} does not exist")
    else:
        # If no tool call is made, print the model's response
        print(response_message.content)
```

> Let's test our music recommendation agent with some lyrics from the song "Nzaza" by Asake.   

```python
Question3(
    lyrics="""
Iyanu yen shock won bakan biti badboy timz
Say wetin go be go be, dem no fit hold me on G
Won le le, won le le, won le le o
From Ojuelegba to Ekate o
I show dem pepper kin to sanle
Because I know my dream, mi o kere oh (kere)
Jazz on jazz but their jazz don cast
Only strong fit to fit survive
.
.
.
    """
)
```

When you run the above code, you should see an output similar to this:

```
============================================
Calling function get_recommendations
Track: Nzaza
Artist: Asake
============================================
Similar Tracks:
Muse by Asake
Ototo by Asake
PRAY by BNXN
Aquafina by Young Jonn
Different Pattern by Seyi Vibez
Pidgin & English by BNXN
New Religion by Olamide
Stubborn (with Asake) by Victony
Stronger by Young Jonn
Cana by Seyi Vibez
============================================

The lyrics you provided are from the song "Nzaza" by Asake. Here are some similar tracks you might enjoy:

1. "Muse" by Asake
2. "Ototo" by Asake
3. "PRAY" by BNXN
4. "Aquafina" by Young Jonn
5. "Different Pattern" by Seyi Vibez
6. "Pidgin & English" by BNXN
7. "New Religion" by Olamide
8. "Stubborn (with Asake)" by Victony
9. "Stronger" by Young Jonn
10. "Cana" by Seyi Vibez

Let me know if there's anything else you'd like to explore!
```

✅ Voila, you've now built a music recommendation agent that uses OpenAI's GPT-4 model to identify songs from lyrics and the Last.fm API to fetch similar tracks. 
We've explored the basic foundation that makes up even the most complex agentic workflows. We've seen how large language models and APIs can work together to create practical, real-world applications.

Fun fact: One of these recommendations, *PRAY*, quickly became a favorite! 

---

References:

* OpenAI's cookbook https://github.com/openai/openai-cookbook/blob/

* LastFM's API Documentation https://www.last.fm/api/show/track.getSimilar


[Code - Colab notebook ↗](https://colab.research.google.com/drive/1xOjYWzdT2IgSeCLIgRsfLBgeQwXqwCng?usp=sharing) 

