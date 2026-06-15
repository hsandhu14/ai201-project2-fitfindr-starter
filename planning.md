# FitFindr — planning.md

> Complete this document before writing any implementation code.
> Your spec and agent diagram are what you'll use to direct AI tools (Claude, Copilot, etc.) to generate your implementation — the more specific they are, the more useful the generated code will be.
> Your planning.md will be reviewed as part of your submission.
> Update it before starting any stretch features.

---

## Tools

List every tool your agent will use. For each tool, fill in all four fields.
You must have at least 3 tools. The three required tools are listed — add any additional tools below them.

### Tool 1: search_listings

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
search_listings searches the mock secondhand listings dataset for items that match the user's requested item description, size, and maximum price. It filters listings using fields like title, description, category, style tags, size, price, colors, brand, and platform, then returns the best matching listings sorted by relevance.

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- description (str): The item or style the user is looking for, such as "vintage graphic tee" or "black denim jacket".
- size (str): The requested clothing size, such as "S", "M", "L", or "XL". This can be used to filter listings by the listing's size field.
- max_price (float): The user's maximum budget for the item. Listings with a price higher than this value should not be returned.

**What it returns:**
<!-- Describe the return value — what fields does a result contain? -->
The tool returns a list[dict] of matching listing dictionaries sorted from most relevant to least relevant. Each dictionary contains information about one listing, including id, title, description, category, style_tags, size, condition, price, colors, brand, and platform. The agent uses the first item in the list as the selected thrift item for the next tool call.

**What happens if it fails or returns nothing:**
<!-- What should the agent do if no listings match? -->
If no listings match the query, the tool returns an empty list instead of crashing. The agent checks for an empty result before moving forward. If the list is empty, the agent should not call suggest_outfit. Instead, it should tell the user that no listings were found and suggest a specific change, such as raising the budget, removing the size filter, or searching for a broader description.

---

### Tool 2: suggest_outfit

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
suggest_outfit takes the selected thrift listing from search_listings and compares it with the user's wardrobe to create one or more complete outfit suggestions. It explains how the new item can be styled with existing pieces, including bottoms, shoes, layers, and accessories when possible.

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
-  new_item (dict): The selected listing dictionary from search_listings. It includes fields such as title, category, style_tags, size, price, colors, brand, and platform.
-  wardrobe (dict): A dictionary representing the user's wardrobe. It contains an items list, where each wardrobe item has fields such as name/title, category, colors, style tags, and other item details from the wardrobe schema.

**What it returns:**
<!-- Describe the return value -->
The tool returns a non-empty str containing one or two complete outfit recommendations. The recommendation should mention the selected thrift item and specific wardrobe pieces when available. It should include a short explanation of the overall style or vibe, such as vintage streetwear, casual grunge, or clean everyday wear.

**What happens if it fails or returns nothing:**
<!-- What should the agent do if the wardrobe is empty or no outfit can be suggested? -->
If the wardrobe is empty or only has minimal items, the tool should still return useful general styling advice instead of crashing. If no specific wardrobe pieces are available, the agent should explain that it is giving general styling suggestions and recommend common pieces that would pair well with the new item.

---

### Tool 3: create_fit_card

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
create_fit_card turns the outfit suggestion into a short, shareable fit card or caption. The output should sound like a casual outfit post rather than a formal product description.

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- outfit (str): The outfit suggestion returned by suggest_outfit.
- new_item (dict): The selected thrift listing dictionary from search_listings, including the item title, price, platform, condition, colors, and style tags.

**What it returns:**
<!-- Describe the return value -->
The tool returns a str that is 2–4 sentences long and could be used as an Instagram, TikTok, or outfit-of-the-day caption. The fit card should naturally mention the thrifted item, the price, the platform, and the overall outfit vibe.

**What happens if it fails or returns nothing:**
<!-- What should the agent do if the outfit data is incomplete? -->
If the outfit input is missing, empty, or incomplete, the tool should return a clear error message instead of crashing. The agent should tell the user that a fit card cannot be created until an outfit suggestion exists, then either retry suggest_outfit or ask the user for more wardrobe details.

---

### Additional Tools (if any)

<!-- Copy the block above for any tools beyond the required three -->
For the required version, I will only implement the three required tools first. After the required workflow works, I may add retry logic with fallback or a price comparison tool as a stretch feature.

---

## Planning Loop

**How does your agent decide which tool to call next?**
<!-- Describe the logic your planning loop uses. What does it look at? What conditions change its behavior? How does it know when it's done? -->
The agent stores all important information in a session state dictionary and uses conditional logic to decide the next tool call.

The agent starts with the user's natural language query and extracts or receives three search inputs: description, size, and max_price.
If state["search_results"] is empty or missing, the agent calls search_listings(description, size, max_price).
After search_listings returns, the agent checks the result:
If the result is an empty list, the agent stores an error message in state and returns early with a helpful message. It does not call suggest_outfit with empty input.
If the result contains listings, the agent sets state["selected_item"] = search_results[0].
If state["selected_item"] exists and state["outfit"] is missing, the agent calls suggest_outfit(state ["selected_item"], wardrobe).
After suggest_outfit returns, the agent checks the result:
If the outfit suggestion is empty, the agent stores an error message and returns a message asking for more wardrobe information.
If the outfit suggestion is non-empty, the agent stores it as state["outfit"].
If state["outfit"] exists and state["fit_card"] is missing, the agent calls create_fit_card(state["outfit"], state["selected_item"]).
After create_fit_card returns, the agent stores the result as state["fit_card"].
The loop is complete when the state contains a selected item, an outfit suggestion, and a fit card. The agent then returns all three pieces to the user.

---

## State Management

**How does information from one tool get passed to the next?**
<!-- Describe how your agent stores and accesses state within a session. What data is tracked? How is it passed between tool calls? -->
The agent uses a session state dictionary to store values created during the interaction. This prevents the user from having to re-enter information between tool calls.

The session state tracks:

state = {
    "user_query": "",
    "description": "",
    "size": "",
    "max_price": None,
    "wardrobe": {},
    "search_results": [],
    "selected_item": None,
    "outfit": "",
    "fit_card": "",
    "errors": []
}

After search_listings runs, the returned list is stored in state["search_results"]. The first listing is stored as state["selected_item"]. That exact dictionary is passed into suggest_outfit.

After suggest_outfit runs, the returned outfit string is stored in state["outfit"]. That exact outfit string is passed into create_fit_card.

After create_fit_card runs, the returned caption is stored in state["fit_card"]. The final user response is built from state["selected_item"], state["outfit"], and state["fit_card"].

---

## Error Handling

For each tool, describe the specific failure mode you're handling and what the agent does in response.

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | The agent says: "I couldn't find any listings that matched that description, size, and budget. Try raising the budget, using a broader search term, or removing the size filter." The agent stops and does not call suggest_outfit. |
| suggest_outfit | Wardrobe is empty | The agent says: "I don't have enough wardrobe items to build a specific outfit from your closet, so I'll give general styling advice instead." The tool returns general outfit ideas using common pieces. |
| create_fit_card | Outfit input is missing or incomplete | The agent says: "I couldn't create a fit card because there is no complete outfit suggestion yet. Try generating an outfit first or adding more wardrobe details." |

---

## Architecture

<!-- Draw a diagram of your agent showing how the components connect:
     User input → Planning Loop → Tools (search_listings, suggest_outfit, create_fit_card)
                                                                          ↕
                                                                   State / Session
     Show what triggers each tool, how state flows between them, and where error paths branch off.
     ASCII art, a Mermaid diagram (https://mermaid.js.org/syntax/flowchart.html), or an embedded
     sketch are all fine. You'll share this diagram with an AI tool when asking it to implement
     the planning loop and each individual tool. -->
     
     User query 
     | v Planning Loop 
     |
     | extracts description, size, max_price, wardrobe 
     v 
     search_listings(description, size, max_price) 
     | results == [] 
     |  +---------------------> Error response: 
     | "No listings found. Try a broader search, 
     | higher budget, or different size." 
     | END 
     | results == [item, ...] 
     | 
     v 
     Session State: 
     selected_item = results[0] 
     search_results = results 
     | 
     v suggest_outfit(selected_item, wardrobe) 
     | 
     | wardrobe empty 
     +---------------------> General styling advice returned 
     | 
     | wardrobe has items
     v 
     Session State: 
     outfit = outfit_suggestion 
     | 
     v 
     create_fit_card(outfit, selected_item) 
     | 
     | outfit missing 
     +---------------------> Error response: 
     | "Cannot create fit card without outfit." 
     | END 
     | 
     v 
     Session State: 
     fit_card = caption 
     | 
     v 
     Final response to user: 
     - selected thrift listing 
     - outfit suggestion 
     - shareable fit card

---

## AI Tool Plan

<!-- For each part of the implementation below, describe:
     - Which AI tool you plan to use (Claude, Copilot, ChatGPT, etc.)
     - What you'll give it as input (which sections of this planning.md, your agent diagram)
     - What you expect it to produce
     - How you'll verify the output matches your spec before moving on

     "I'll use AI to help me code" is not a plan.
     "I'll give Claude my Tool 1 spec (inputs, return value, failure mode) and ask it to implement
     search_listings() using load_listings() from the data loader — then test it against 3 queries
     before trusting it" is a plan. -->

**Milestone 3 — Individual tool implementations:**
I will use ChatGPT to help implement the three required tools one at a time. For search_listings, I will give ChatGPT the Tool 1 section of this planning document and the function signature from tools.py. I expect it to produce Python code that uses load_listings(), filters by description, size, and max_price, sorts matching listings by relevance, and returns an empty list when nothing matches.
Before using the generated code, I will check that it does not crash on no results, that it uses the existing data loader instead of rewriting file loading, and that the return value is a list of listing dictionaries. I will test it with at least three searches: one normal query, one query with a strict price limit, and one query that should return no results.

For suggest_outfit, I will give ChatGPT the Tool 2 section, the wardrobe schema, and the function signature from tools.py. I expect it to produce code that accepts a selected listing and wardrobe dictionary, handles an empty wardrobe, and returns a non-empty string with outfit suggestions. I will verify that the output uses the selected item and does not crash when get_empty_wardrobe() is passed in.
For create_fit_card, I will give ChatGPT the Tool 3 section and the function signature from tools.py. I expect it to produce code that creates a 2–4 sentence caption using the outfit suggestion and selected item. I will verify that it handles an empty outfit string with a clear error message and creates different captions for different item/outfit inputs.

**Milestone 4 — Planning loop and state management:**
I will use ChatGPT to help implement the planning loop in agent.py. I will give it the Planning Loop, State Management, Error Handling, and Architecture sections of this planning document. I expect it to produce a loop or equivalent conditional control flow that stores search_results, selected_item, outfit, and fit_card in a session state dictionary.

Before using the generated code, I will compare each branch to the planning loop spec. I will verify that the agent stops early if search_listings returns no results, that the selected item from search is passed into suggest_outfit, and that the outfit from suggest_outfit is passed into create_fit_card. I will test the full workflow with one happy-path query and one failure-path query.

---

## A Complete Interaction (Step by Step)

Write out what a full user interaction looks like from start to finish — tool call by tool call. Use a specific example query.

**Example user query:** "I'm looking for a vintage graphic tee under $30. I mostly wear baggy jeans and chunky sneakers. What's out there and how would I style it?"

**Step 1:**
<!--  -->
The agent calls search_listings (description="vintage graphic tee", size="M", max_price=30.0). The tool searches the listings dataset for matching items based on the description, size, and price limit. The tool returns several matching listings, and the agent stores the highest-ranked result as selected_item.

**Step 2:**
<!--  -->
The agent calls suggest_outfit (new_item=selected_item, wardrobe=get_example_wardrobe()). The selected listing from Step 1 is passed directly into the tool without the user having to re-enter it. The tool analyzes the user's wardrobe and returns a complete outfit recommendation that includes the new thrifted item.

**Step 3:**
<!--  -->
The agent stores the outfit recommendation as outfit and calls create_fit_card(outfit=outfit, new_item=selected_item). The outfit from Step 2 is passed directly into the tool without re-entry. The tool generates a short social-media-style caption and outfit summary based on the selected item and outfit combination.

**Final output to user:**
<!--  -->
Found: Faded Band Tee — $22, Good Condition, Depop.

Suggested Outfit: Pair the graphic tee with your baggy jeans and chunky sneakers for a relaxed vintage streetwear look. Add a lightweight overshirt or denim jacket for layering.

Fit Card: "Thrifted this faded band tee for $22 and it pairs perfectly with my baggy denim and chunky sneakers. Easy vintage vibes without breaking the budget."
