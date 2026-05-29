# Daily Intelligence AI Bot V1

***

## Goal
To create an AI bot that parses cybersecurity news of the day, ranks the articles based on a keyword list and scoring system and generates a professional looking markdown file. This is V1 only just to see how it would work. Future versions will have this scheduled and Telegram support.

## Setup
For this I am just using a local Ollama model, Qwen. To set that up:
```
curl -fsSL https://ollama.com/install.sh | sh
```
Check that Ollama is installed
```
ollama --version
```
Check if it autostarted after install
```
systemctl status ollama
```
If not, then:
```
sudo systemctl enable ollama
sudo systemctl start ollama
```
Install Qwen
```
ollama pull qwen2.5:7b
```
Do some Pip installs
```
pip install ollama feedparser markdown weasyprint requests beautifulsoup4 newspaper3k lxml_html_clean
```
Make your project folder structure
```
mkdir -p ~/cyberintel/{feeds,reports,scripts,logs,db}
cd ~/cyberintel
```
In the scripts folder make a new script
```
nano ~/cyberintel/scripts/daily_report.py
```
Here is the script
```
import os
import feedparser
import ollama
from newspaper import Article
from datetime import datetime

BASE_DIR = os.path.expanduser("~/cyberintel")
REPORT_DIR = os.path.join(BASE_DIR, "reports")

feeds = [
    "https://feeds.feedburner.com/TheHackersNews",
    "https://krebsonsecurity.com/feed/",
    "https://www.bleepingcomputer.com/feed/",
]

articles = []

# -----------------------------
# FETCH RSS ARTICLES
# -----------------------------

for feed_url in feeds:

    feed = feedparser.parse(feed_url)

    for entry in feed.entries[:5]:

        try:

            article_url = entry.link

            news_article = Article(article_url)
            news_article.download()
            news_article.parse()

            full_text = news_article.text

            if len(full_text) < 500:
                continue

            articles.append({
                "title": entry.title,
                "link": article_url,
                "content": full_text
            })

            print(f"[+] Pulled: {entry.title}")

        except Exception as e:
            print(f"[!] Failed article: {e}")

# -----------------------------
# KEYWORD SCORING
# -----------------------------

keywords = {
    "critical": 5,
    "ransomware": 5,
    "zero-day": 5,
    "cve": 4,
    "breach": 4,
    "exploit": 3,
    "phishing": 2,
    "malware": 2,
    "vulnerability": 3,
    "backdoor": 4
}

for article in articles:

    score = 0

    content_lower = article["content"].lower()

    for keyword, weight in keywords.items():

        if keyword in content_lower:
            score += weight

    article["score"] = score

# -----------------------------
# SORT BY PRIORITY
# -----------------------------

articles = sorted(
    articles,
    key=lambda x: x["score"],
    reverse=True
)

# -----------------------------
# BUILD REPORT
# -----------------------------

report = f"# Daily Cybersecurity Intelligence Report\n\n"
report += f"Generated: {datetime.now()}\n\n"

for article in articles:

    print(f"[+] Analyzing: {article['title']}")

    prompt = f"""
You are a cybersecurity threat intelligence analyst.

Analyze this article and return markdown output with:

# Severity
# Executive Summary
# Technical Details
# Business Impact
# Recommended Actions
# Key Indicators
# Mentioned CVEs

Article:
{article['content'][:12000]}
"""

    try:

        response = ollama.chat(
            model='qwen2.5:7b',
            messages=[
                {
                    'role': 'user',
                    'content': prompt
                }
            ]
        )

        summary = response['message']['content']

    except Exception as e:

        summary = f"Analysis failed: {e}"

    report += f"\n\n"
    report += f"# {article['title']}\n\n"
    report += f"Priority Score: {article['score']}\n\n"
    report += f"Source: {article['link']}\n\n"
    report += summary
    report += "\n\n"

# -----------------------------
# SAVE REPORT
# -----------------------------

filename = os.path.join(
    REPORT_DIR,
    f"report_{datetime.now().strftime('%Y%m%d')}.md"
)

with open(filename, "w") as f:
    f.write(report)

print(f"\n[+] Report saved: {filename}")
```

## Script explanation
Here is a mental breakdown of what its doing:
```
RSS Feeds
   ↓
Download Articles
   ↓
Extract Clean Text
   ↓
Score Importance
   ↓
Sort by Priority
   ↓
Send to Qwen
   ↓
Generate Intelligence Report
   ↓
Save Markdown File
```

### Imports
```
import os
import feedparser
import ollama
from newspaper import Article
from datetime import datetime
```
These are Python libraries/modules.
1. import os - The os module helps Python interact with the operating system. You use it for:
* file paths
* directories
* environment variables
2. import feedparser - This library reads RSS feeds. As an example, https://krebsonsecurity.com/feed/ has article titles, links, publish dates and summaries. Feedparser converts that XML into Python objects.
3. import ollama - I installed Qwen model via Ollama. This is your local AI interface. It lets Python talk to your locally running Qwen model. Think of it like: Python → Ollama API → Qwen Model.
4. from newspaper import Article - RSS feeds only give snippets. With newspaper3k it downloads webpages, removes ads/navigation and extracts clean article text. Without this, your summaries would be weak.
5. from datetime import datetime - Used for things like timestamps, filenames, report dates, etc. For example, datetime.now() returns current date/time.

### Project Paths
```
BASE_DIR = os.path.expanduser("~/cyberintel")
REPORT_DIR = os.path.join(BASE_DIR, "reports")
```
1. BASE_DIR - The ~ means the current user home directory (in Linux). In this example I made a folder called cyberintel in my Ubuntu home directory.
2. REPORT_DIR - This combines /home/ubuntu/cyberintel with the folder 'reports' to create: /home/ubuntu/cyberintel/reports

### RSS Feed List
```
feeds = [
    "https://feeds.feedburner.com/TheHackersNews",
    "https://krebsonsecurity.com/feed/",
    "https://www.bleepingcomputer.com/feed/",
]
```
This is a Python list. Each item is an RSS feed URL and you can add as many as you want.

### Empty Article Container
```
articles = []
```
This creates an empty list. You’ll fill it with article data later. Think of it conceptually like: articles = storage bucket. Then later it will become:
```
[
  {
    "title": "...",
    "link": "...",
    "content": "...",
    "score": 10
  }
]
```

### RSS Feed Loop
```
for feed_url in feeds:
```
This means: for every RSS feed in the list... Python loops through each URL one at a time. So it will start with TheHackerNews, then move to krebsonsecurity, then bleeping computer. feeds: is the list we made earlier with all the RSS feed URLs.

### Parse RSS Feed
```
feed = feedparser.parse(feed_url)
```
You recall we imported the feedparser library at the top. This downloads the RSS XML and parses it. Now feed.entries contains all articles.

### Article Loop
```
for entry in feed.entries[:5]:
```
This loops through the first 5 articles. The [:5] is called slicing. When it says [0:5] what that means is item 0 through 4, because Python uses a zero based method of counting. This is to limit the article amount. You limit article count so processing stays manageable Qwen (LLM Model) doesn't get overloaded. 

### Error Handling
```
try:
```
This begins a protected block. If something fails like a broken website, timeout, malformed article, etc. your script won't crash. You will see these blocks multiple times for different sections of code.


### Article URL
```
article_url = entry.link
```
Gets the actual webpage URL from the RSS feed. Recall our For Loop earlier (for entry in feed.entries[:5]:) For example it will look at the BleepingComputer RSS Feed and pull the URL https://www.bleepingcomputer.com/news/security/...

### Download Article
```
news_article = Article(article_url)
news_article.download()
news_article.parse()
```
1. Line 1 creates a newspaper article object.
2. Line 2 downloads the webpage HTML.
3. Line 3 extracts
* article body
* removes ads
* removes menus
* removes junk

Now you have clean text. Recall our import at the beginning of the script, from newspaper import Article. This is what that library is doing.

### Extract Text
```
full_text = news_article.text
```
This gets the cleaned article text. This is what gets sent to Qwen.

### Filter Tiny Articles
```
if len(full_text) < 500:
    continue
```
Some articles fail extraction, are too short or are garbage pages. This skips low-quality content.
1. len(full_text) - Counts characters. For example, len("hello") returns '5' as there is 5 characters.
2. continue - Means skip this article and move to the next one.

### Store Article
```
articles.append({
    "title": entry.title,
    "link": article_url,
    "content": full_text
})
```
This adds a dictionary into the articles list. Dictionary = key/value structure. Like:
```
{
  "title": "...",
  "link": "...",
  "content": "..."
}
```

### Logging
```
print(f"[+] Pulled: {entry.title}")
```
This displays progress. Some examples of what you would see when you run the script:
```
[+] Pulled: New Russian-Linked GREYVIBE Targets Ukraine with AI-Powered Cyberattacks
[+] Pulled: What 2,000 Exposed Vibe-Coded Apps Reveal About the Limits of Most Security Stacks
[+] Pulled: Malicious Sicoob NuGet Steals Banking Credentials as npm Packages Target Cloud Secrets
```
This is useful for debugging as you can see what your script is doing in realtime. 

### Exception Handling
```
except Exception as e:
    print(f"[!] Failed article: {e}")
```
If something breaks like a bad webpage, timeout, parsing error, etc. the script logs it instead of crashing. You will note it is part of the 'try' block. This is why you may see it called a try/except block. As we tell it to put the error in a variable called 'e', the print statement adds that variable to the error text that we choose. 


### Keyword Scoring
```
keywords = {
    "critical": 5,
    "ransomware": 5,
    "zero-day": 5,
    "cve": 4,
}
```
This is a dictionary. Each keyword has an importance weight. 


### Score Articles
```
for article in articles:
```
Loop through all stored articles.

### Initialize Score
```
score = 0
```
Every article starts with a score of zero.

### Lowercase Content
```
content_lower = article["content"].lower()
```
Makes searching easier. Without this:
1. Ransomware
2. ransomware
3. RANSOMWARE

Would all behave differently.


### Keyword Matching
```
for keyword, weight in keywords.items():
```
This loops through dictionary pairs. For example:
```
keyword = ransomware
weight = 5
```


### Check for Keyword
```
if keyword in content_lower:
```
Checks whether the article contains the keyword.


### Increase Score
```
score += weight
```
Adds points to it. For example, all articles start with zero, so if this keyword has a weight of 5, then 0 + 5 = 5


### Save Score
```
article["score"] = score
```
Stores score inside article dictionary. Now article contains (if your score was 12, aggregating multiple keywords):
```
{
  "title": "...",
  "content": "...",
  "score": 12
}
```


### Sort by Priority
```
articles = sorted(
    articles,
    key=lambda x: x["score"],
    reverse=True
)
```
This sorts articles with the highest score first
1. lambda - Tiny anonymous function. Equivalent to: use article["score"] as the sorting value
2. reverse=True - Descending order. Without it the lowest scores appear first


### Build Report
```
report = f"# Daily Cybersecurity Intelligence Report\n\n"
```
Creates markdown content. # means markdown heading for like a Title style. ## is a little smaller. 


### Add Timestamp
```
report += f"Generated: {datetime.now()}\n\n"
```
Adds report generation date. 


### AI Analysis Loop
```
for article in articles:
```
Now each article gets analyzed by Qwen (or whatever model you chose)


### Prompt Engineering
```
prompt = f"""
You are a cybersecurity threat intelligence analyst....
```
You’re instructing the AI:
1. Its role
2. output format
3. analysis type

Better prompts will give you dramatically better results. 


### Truncate Content
```
{article['content'][:12000]}
```
Limits article size. Important because LLMs have context limits and giant prompts slow inference. 


### Send to Qwen
```
response = ollama.chat(
```
Python calls Ollama API. You recall we imported the Ollama library.


```
model='qwen2.5:7b'
```
Specifies which local model to use

```
messages=[
    {
        'role': 'user',
        'content': prompt
    }
]
```
This mimics ChatGPT conversation format.


### Extract AI Output
```
summary = response['message']['content']
```
Gets actual text generated by Qwen. 


### Add to Report
```
report += f"# {article['title']}\n\n"
```
Adds article section to markdown report.


### Save Report
```
with open(filename, "w") as f:
```
Creates/writes file. 'w' in this case means 'write'


### Write Data
```
f.write(report)
```
Saves markdown text to disk.


### Completion Message
```
print(f"\n[+] Report saved: {filename}")
```
Final success message.


## RSS Feeds List
Here is some RSS Feeds you can add
```
https://thehackernews.com/feeds/posts/default
https://www.bleepingcomputer.com/feed/
https://www.darkreading.com/rss.xml
https://feeds.feedburner.com/securityweek
https://threatpost.com/feed/
https://krebsonsecurity.com/feed/
https://www.cisa.gov/cybersecurity-advisories/all.xml
https://www.cisa.gov/news-events/cybersecurity-news.xml
https://nvd.nist.gov/feeds/xml/cve/misc/nvd-rss.xml
https://www.ic3.gov/Home/RSS
https://www.cisecurity.org/feed/advisories
https://www.mandiant.com/resources/blog/rss.xml
https://unit42.paloaltonetworks.com/feed/
https://blog.talosintelligence.com/feeds/posts/default
https://www.crowdstrike.com/blog/feed/
https://www.sentinelone.com/labs/feed/
https://blog.malwarebytes.com/feed/
https://securelist.com/feed/
https://www.exploit-db.com/rss.xml
https://seclists.org/rss/fulldisclosure.rss
https://isc.sans.edu/rssfeed.xml
https://msrc.microsoft.com/blog/feed
https://www.zerodayinitiative.com/blog?format=rss
https://badpackets.net/feed/
https://therecord.media/feed/
```

## Summary
After running the script you should see a markdown file in the reports folder. 













































