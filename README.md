# ZenQuotes 'On This Date' API ( hereinafter referred to as "OTD") + API Request Plugin for Obsidian (hereinafter referred to as "APIR") = Daily Historical Events rendered on an Obsidian Daily Note #
## Inspiration ##
- In my attempts to craft a daily note template that was both reflective and informative, I wanted to not only insert daily quotes, but also historical events.
- The solution was found via [ZenQuotes](https://www.zenquotes.io).
  - The OTD endpoint is "https://today.zenquotes.io/api" with no arguments, which generates the OTD events for its current server time which is UTC.
    - This endpoint is all that is needed for the daily note. 
      - OTD also includes birthdays and deaths, but it took up too much space for my taste. I will include it herein as an option.
## Solution ##
- In order to have this solution work properly, I needed to request the API for OTD, parse its resulting JSON for the events, and then format the output for use in the daily note.
  - However, the largest issue that I dealt with was the constant refreshing of the API, so I included a bootstrapped solution to cache the JSON data, and then render it in a useable format.
  - Once that hurdle was surpassed, I realized, that it would simply keep the cache for the first day that I utilized this script, and simply output that EVERY DAY thereafter, so I needed it to auto-update based on the date it was called in the Daily Note.
    - This was a bit more difficult, but I found a working solution in creating a callout at the bottom of my daily note that utilizes the API Request plugin and auto-updates (showing its output as the current date), but this information is read by the dataview script to output the daily events.
    - NGL, it's kinda janky, because if you edit daily notes from a previous date, it will update them to include the current date events, but it works.
- Feel free to make pull requests and I'll give credit where it's due for a cleaner solution.
## Caveats ##
- Local Storage in APIRequest uses the `req-uuid` flag.
  - The responses are stored in the browser's localStorage and called by that UUID.
  - `auto-update` re-fetches the API request every time the note loads, which is the opposite of what I wanted.
    - As a result, the caching logic needed to be handled manually in a separate dataviewjs block that checks whether today's date matches what's already stored.
- My initial approach was mistaken because `auto-update` in APIR eseentially fetches OTD again every time the note loads, and I thought it helped with a daily refresh. 
  - It does the opposite of caching, so using it alongside a `dataviewjs` cache check created a race condition.
    - The `req` block would always fetch regardless of what the `dataviewjs` block did.
      - The caching logic was being split across two blocks that were ultimately fighting each other.
- After beating my head against the wall, and then re-reading the `dataviewjs` and sparse `APIR` docs, I thought that I could handle everything in a single dataviewjs block.
  - My rationale was that I could have the code block (a) check the cache, (b) call the API (only if it was from the previous day or "stale", (c) store the result, and then (c) render the output; all without involving the `req` block.
    - The APIR `req` block has no conditional logic capabilities, so it cannot effectively make the caching choices.
    - `dataviewjs` can call `fetch()` and control the cache logic.
    - As a result, `localStorage.getItem("req-otd")` was returning as null because the APIR block wasn't populating properly.
    - The `hidden` property in APIR was hiding the rendered output of the entire note section, not just the `req` block.
      - This required me to put the `req` block in a separate section to act as a "data source," as well as take the `req` block which has to be visible but unnecessarily formatted or styled.
        - As a result, I put the `req` block at the bottom of my daily note template, in a callout, so it's out of my way and semi-hidden.
          - The `req` block without any formatting flags just outputs raw JSON, which is not ideal, so I only pulled a single field; the date string instead of the entire JSON block.
            - This creates a single line at the bottom of my note; the rendered date is unavoidable because I am using APIR for the OTD call.
            - Everything else is stored in `localStorage` under `req-otd` and can be read by the `dataviewjs` block.

- At this point, the `dataviewjs` block was still reading the incorrect cache.
  - The `req` block was still executing and storing data via the `req-otd` variable, but the `dataviewjs` block wasn't locating it to format and render.
    - In attempting to troubleshoot, I found out that Obsidian has neat methods for inspecting raw JSON and cached objects.
      - As a result, I realized that the data in the stored JSON was there, so it was an issue with what I was declaring as `data` in JSON.
        - The resulting JSON output of "data" in ZenQuotes' response is structured differently than I had assumed from my cursory glance of the output. Meaning that the paths were incorrect.
            - The ZenQuotes response looks like this at the top level:
              ````
              ```json{
                "info": "OnThisDay API",
                "date": "March23",
                "updated": "1773520745",
                "data": {
                  "Events": [...],
                  "Births": [...],
                  "Deaths": [...]
                }
              }
              ```
              ````
            - My initial script was written as `const data = JSON.parse(localStorage.getItem(cache_key));` (yes, I know I am using snake case rather than camel case, python habits die hard), which was linking the entire response object to `data`, meaning the top-level object of `data`.
              - As a result, `Events` would be looking for the Events at the top level.
              - However, `Events` does not live at the top-level of `data` as I had mistakenly assumed. I needed to go one-level deeper to the inner data within the `data` object. 
                - The "Events" section (and at the time Births and Deaths) are in fact found at "data.Events", "data.Births", and "data.Deaths". Once I changed the script to assign these objects to the variable data, the script worked as intended.
                  - I found a similar line on StackOverflow that made a dictionary variable and used a method to parse the JSON object and link that to the cache variable.
                    - `const parsed_data = JSON.parse(localStorage.getItem(cache_key));` + `const data = parsed_data.data`
                      - Refactoring this resulted in: `const { data } = JSON.parse(localStorage.getItem(cache_key)`.


