# Introduction

The story of this project starts in the Udacity Data Scientist Nanodegree. The Nanodegree has a simple course about web development using Bootstrap, Flask and JQuery. 

They provided an ungraded project in which you create a dashboard webapp using Flask that fetched data and visualizes it from the WorldBank API.

But I really wanted to do something unique in this project, and I thought that doing the project this way wouldn't teach me anything worthwhile, so I kept digging and digging for ideas that might interest me and have the right mix of challenge and ease that won't get me discouraged.

I had been interested in NLP for a while, but this interest didn't produce any fruitful outcome until now, so I thought that this project should have some NLP in it.

And It finally occurred to me to analyze movie scripts and compare them with each other, specifically of my favorite directors Christopher Nolan and Quentin Tarantino, since they both write their movies' screenplays.

So I started preparing for this project and searching for ways to get the data, and while exploring the things that can be done, the most interesting thing that I found was analyzing the network of interactions between the characters in the script.

A network graph consists of nodes that have various sizes, which are connected by edges. What I had in mind was to make a node for each character with its size varying according to the number of dialogues they have in the script, and to trace the edges between these nodes according to the number of interaction between these characters, and it's illustrated in the following image.

<<IMAGE>>
  
  
After successfully making the first network graph, I found that the method could generalize to any script (as all scripts almost have the same anatomy), and so I decided to start a stand-alone project which is a webapp that can show the character network of (almost) any movie. 

The pipeline of the project consisted of the following steps:

1. Getting the data
2. Preparing an app that can make network graphs using movie scripts
3. Deploying the app on the web using heroku
  
  
## Getting the data

This step was really easy as I found [https://imsdb.com/](https://imsdb.com/) which is a database the contains numerous movie scripts in HTML format.  

Using requests and BeautifulSoup I was able to:

1. Fetch and extract all movies relative links in imsdb
2. Loop over the movies and extract their script texts from HTML pages using BeautifulSoup
3. Save the scripts texts and saving a clean version of the movie name for further lookups
  
  
```
  response = requests.get('https://imsdb.com/all-scripts.html')
html = response.text

soup = BeautifulSoup(html, "html.parser")
paragraphs = soup.find_all('p')

movies_files = {}

for p in paragraphs:
    relative_link = p.a['href']
    search_title, save_title, script = get_script(relative_link)
    if not script:
        continue

    # clean save title from any punctuations that prevent creating a filename
    save_title = save_title[:-5].translate(str.maketrans('', '', string.punctuation)) + '.txt'
    script_file = os.path.join(SCRIPTS_DIR, save_title)

    # save script text
    with open(script_file, 'w', encoding='utf-8') as outfile:
        outfile.write(script)

    # save mapping of movie name with script text file
    movies_files[search_title] = script_file

# save mapping to a picke file for easy loading
movies_files_pkl =  open(os.path.join(SCRIPTS_DIR, MOVIES), 'wb')
pickle.dump(movies_files, movies_files_pkl)
```
  
 For more information about cleaning the HTML you can check the github repo.

## Preparing the visualization app

Now this part was the one that interested me the most, how could I extract characters interactions throughout the script? 

I didn't find a perfect answer, but I found one nonetheless.

Let's take a look at the anatomy of a typical script.
  
 
 <<IMAGE>>
  
We can see that a character name is distinct as it's always written in uppercase. We can also see that the only part of the script which is also written in uppercase is the scene heading. 

So what if we split the script into sentences, then we looped over them and identified character names, then appended the following dialogue to them.

Using this approach we would be able to identify the dialogue belonging to each character in the script. And that's exactly what I did.

The condition that I found out was most differentiating between scene headings and character names was the presence of dots and dashes in scene headings. So I used a regular expression that checks for the presence in this in the sentence, alongside being in upper case, to judge whether it's a character name or not.
 
```
def extract_dialogues(script_sents):
    """Function to extract dialogues from tokenized script sentences."""
    pattern_compile = re.compile(r'[.,–!:]')
    dialogues = []
    add_dialogue = False
    for sent in script_sents:
        if not (sent.startswith('(') and sent.endswith(')')):
            if add_dialogue:
                if not (sent.strip().isupper() and not pattern_compile.search(sent)):
                    dialogues.append((character, sent))
                add_dialogue = False
            if (sent.strip().isupper() and not pattern_compile.search(sent))\
               and not (sent.strip().startswith('(') and sent.strip().endswith(')')):
                character = sent.strip()
                add_dialogue = True
    return dialogues
 ```
  
  The next part was finding out through this extracted dialogue, who is talking to whom? 

To be honest, I used the most simple approach that crossed my mind, and that was shifting the dialogue between characters one step backward, so this way if X said something followed by Y saying something, you'd get the X said something to Y, and if X said something after Y, you'd get that Y also said something to X.
  
```
  text = open(movie_file, 'r').read()
text = clean_script_text(text)
sents = sent_tokenize_script(text)
dialogue = extract_dialogues(sents)
dialogue_df = pd.DataFrame(dialogue, columns=['character', 'text'])

# Make list of dialogue exchanged
dialogue_df['character_shifted'] = dialogue_df.character.shift(-1)
```
  
But what if a scene ends with X saying something, then another scene starts with Z saying something, wouldn't that mean that we'd get that X said something to Z?

That's true, but it wouldn't matter that much, because if X regularly interacted with Z throughout the whole script, a wrongly attributed interaction between them wouldn't affect the results that much, and if they never interacted, then this wrongfully attributed interaction would be filtered out in the end of the visualization pipeline as we don't want to show extremely rare interactions between characters, even if they were correct.

Also what if a scene ended with X, then another scene started with X? or what if a scene had X say two sentences back to back?

The easy solution to this was to drop any exchange from a character to itself all together, but still count it into their dialogue count as this would affect the sizes of their node in the network visualization.
  
 
```
# extract character pairs
pairs = dialogue_df[['character', 'character_shifted']].values.tolist()[:-1]

# remove dialogues from one character to themselves and sort exchanges
pairs = ['-'.join(sorted(x)) for x in pairs if x[0] != x[1]]
```
