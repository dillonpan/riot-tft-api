# riot-tft-api
Project using the package "reader" and Riot's API to retrieve and breakdown JSON information

# Project Details:
In this project, we will use a few of Riot's public API and retrieve multiple JSON objects from it. Afterwards, we will breakdown the information we retrieve for our goal in this project. Before we explain our goal, here's a small summary from [Wikipedia](https://leagueoflegends.fandom.com/wiki/Teamfight_Tactics) on what TFT (Teamfight Tactics) is:

"Assemble an army of your favorite champions in Teamfight Tactics, the PVP strategy autobattler from the makers of League of Legends.
In Teamfight Tactics, you’ll draft, deploy, and upgrade from a revolving roster of League of Legends champions in a round-based battle for survival. Devastate with demons, bruise with brawlers, or transform the battle with shapeshifters—the strategy is all up to you."

The game is developed by Riot Games, hence why we will be using their API. Based on the last sentence of the wiki description above, we can see this strategy game includes various possible team traits (demons, brawlers, shapeshifters, etc). The game is an 8 player "last man standing" auto battler. I play this game quite a bit so I've always been curious to see which team traits have historically gotten me closer to 1st place. Let's create a small apllication to do just that.

Note: This API requires an account with Riot and also an API access key that's generated from logging in to their [developer website](https://developer.riotgames.com/). To protect my account, I will substitute certain code with something more general. Specifically the following information (Consider both as "string" variables):
- (My own Riot "Summoners Name") -> (Summoner Name)
- (My API access code) -> (api code)

Here's also the general steps we will be taking with Riot's API for better understanding:
1. Taking our "Summoners Name" to pull some summoner information, specifically our PUUID
2. Take that PUUID to get a list of our recent games played (20 as of this moment)and each of their "Match ID's"
3. Use each Match ID to pull details of the match to analyze and find our team trait for each game.

# Using the Summoner's Name to Pull the PUUID:
So the first thing we need to do is import the "requests" package and provide some information, specifically our generated API key and our Summoner's name (both being hidden for security):

```python
import requests

# information needed
api_key = (api code)
summoner_name = (Summoner Name)
```
When calling a JSON object from an API we usually need a header and a parameter, both in the form of a dictionary. The information we need in both are usually provided by the API website. In this case, we need the following:
```python
# neccesary headers- taken from Riot API website
headers = {
    "Origin": "https://developer.riotgames.com",
    "Accept-Charset": "application/x-www-form-urlencoded; charset=UTF-8",
    "X-Riot-Token": api_key,
    "Accept-Language": "en-US,en;q=0.9",
    "User-Agent": (personal-hidden)
}

# neccesary parameter - also taken from Riot API website
parameters = {"api_key": api_key}
```

Now we can use the get() method under the requests package to retrieve and turn the JSON object in to a variable:
```python
# the link taken from Riot API website
sum_response = requests.get('https://na1.api.riotgames.com/tft/summoner/v1/summoners/by-name/' + summoner_name,
                            headers=headers,
                            params=parameters)
print(sum_response.status_code)
summoner = sum_response.json()
print(summoner)
```
Everytime you send a requests, you receive a HTTP response code back. "200" means successful while other numbers would represent different errors. The information we successfully retrieved was in the form a dictionary. I've replaced the personal values retrieved with the type of that value you would receive.  

200  
{  
    "profileIconId": (int),  
    "name": (string),  
    "puuid": (string),  
    "summonerLevel": (int),  
    "accountId": (string),  
    "id": (string),  
    "revisionDate": (int)  
}  

# Using the PUUID to Pull Match ID's
```python
puuid = summoner['puuid']

# the link taken from Riot API website
matches_response = requests.get('https://americas.api.riotgames.com/tft/match/v1/matches/by-puuid/' + puuid + '/ids?',
                                headers=headers,
                                params=parameters)

print(matches_response.status_code)
match_list = matches_response.json()
print(match_list)
```
The retrieved data this time came back as a list of Match ID's. I've altered the numbers at the end of each match ID:  

200  
[  
    "NA1_1111111111",  
    "NA1_2222222222",  
    "NA1_3333333333",  
    "NA1_4444444444",  
    "NA1_5555555555",  
    "NA1_6666666666",  
    "NA1_7777777777",  
    "NA1_(numbers)",  
    "NA1_(numbers)",  
    "NA1_(numbers)",  
    "NA1_(numbers)",  
    "NA1_(numbers)",  
    "NA1_(numbers)",  
    "NA1_(numbers)",  
    "NA1_(numbers)",  
    "NA1_(numbers)",  
    "NA1_(numbers)",  
    "NA1_(numbers)",  
    "NA1_(numbers)",  
    "NA1_(numbers)",  
]  

# Taking Match ID's to Pull Match Details
So we have the list of match ID's now but there's no actual details of each match. What we can do is create a dictionary where each of the match ID's will be the key and the details for each match can be connected value:
```python
match_dict = {}
# the link taken from Riot API website
for match in match_list:
    game_response = requests.get('https://americas.api.riotgames.com/tft/match/v1/matches/' + match,
                                 headers=headers,
                                 params=parameters)
    game_details = game_response.json()
    match_dict[match] = game_details
```
The below is what was pulled as "game_details". It's a bunch of dictionaries and lists within other dictionaries and lists. It may seem a bit overwhelming at first but we can take it step by step:
![image](https://user-images.githubusercontent.com/57373723/69102744-284f4d00-0a18-11ea-8b67-594a01f08b57.png)
![image](https://user-images.githubusercontent.com/57373723/69102132-5fbcfa00-0a16-11ea-8200-669fc6ee102a.png)

You can see that the return value is a MatchDto library but within it is InfoDto and MetadataDto, both being dictionaries with other lists/dictionaries within them. From looking over all the details in each match, it seems like we want the TraitDto's. Remember, in each match there are details of all 8 participants and we only want ours. The order to reach this seems to be MatchDto -> info -> participants -> our ParticipantDto -> traits. Seems simple, right?

# Fetching our Trait Details from "game_details"
Let's first create a dictionary to hold the traits. We can create it where the key will be each trait and their corresponding values can be a list of final placements in games. Remember, we have 20 matches to go through in match_dict so we should create a loop that can go through each of them. The next part if a bit complicated so I will break it down in sections then provide the full code at the end.

```python
trait_dict = {}
```
For the below snippet of code, we are going through the dictionaries and lists of each of the 20 matches. For each match, we're jumping in to the list of ParticipantDto's under "participants" in InfoDto. From there, we're going to loop through that list until we find the ParticipantDto with our unique PUUID:
```Python
# For each match in our match dict
for match in match_dict:
    # Go into the details to grab the list of participants
    participants = match_dict[match]['info']['participants']
    # Go through each participant to find yourself
    for ParticipantDto in participants:
        if ParticipantDto['puuid'] == puuid:
```
Here's a small visual that hopefully helps visualize the process of getting to our list of traits for each game:
![image](https://user-images.githubusercontent.com/57373723/69104614-bed23d00-0a1d-11ea-81fd-49f6117743fa.png)

We are now entering the list of TraitDto's for our PUUID (for one game). This is what is in each TraitDto:
![image](https://user-images.githubusercontent.com/57373723/69104900-ac0c3800-0a1e-11ea-8ecc-18d103f3fe7c.png)

Before we understand the snippet below, I need to explain a little more about traits in the game. Each character generally has 1 or 2 traits which link them to other units. With enough different units with the same trait are on the board, you get a "trait bonus". There's also various tiers for each trait type (Generally either 2,4,6 or 3,6). For example, I get a trait bonus if I have two glacial units but an even greater bonus if I can get four glacial units on the board at the same time. Your team composition is mainly based on units with similiar traits. There are times where you are going for the highest tier of one trait but you coincidentally end up also receiving the lowest bonus for another tier, because each unit generally has 2 traits connected to them.

We only want the traits which we were actually focusing on for our team composition. But how can we seperate which traits we were focusing on from those that were an accident in each game? We can use the tier_total & tier_current sections of the details to help is create some filtering logic. Here's an example of what that section looks like:

![image](https://user-images.githubusercontent.com/57373723/69105933-ae23c600-0a21-11ea-91be-e39286fc98f5.png)

Let's set up the following logic. If our current_tier for that trait is 0, we obviously didn't build our team composition with that trait in mind. If the difference between the maximum possible tier for a trait and our current_tier is 2 levels or more, its most likely we acquired that current_tier bonus unintentionally. Thus, if we run in to these two situations, we won't do anything:
```python
            for trait in ParticipantDto['traits']:
                if trait['tier_current'] == 0:
                    pass
                elif trait['tier_total'] - trait['tier_current'] >= 2:
                    pass
```

If our trait does not fulfill the above rules, we can consider it a main trait that we did prioritize in our team composition. Thus, we can add it to our trait_dict. Again, we want a list of final placements as our value to each key(trait):
```python
                else:
                    # if tier prioritized, we either append the game result to the list under that tier or create a new
                    # list if trait didnt exist in the dictionary
                    if trait['name'] in trait_dict:
                        trait_dict[trait['name']].append(float(ParticipantDto['placement']))
                    else:
                        trait_dict[trait['name']] = [float(ParticipantDto['placement'])]
```

Combining the snippets in to a single block of code:
```python
for match in match_dict:
    participants = match_dict[match]['info']['participants']
    for ParticipantDto in participants:
        if ParticipantDto['puuid'] == puuid:
            for trait in ParticipantDto['traits']:
                if trait['tier_current'] == 0:
                    pass
                elif trait['tier_total'] - trait['tier_current'] >= 2:
                    pass
                else:
                    if trait['name'] in trait_dict:
                        trait_dict[trait['name']].append(float(ParticipantDto['placement']))
                    else:
                        trait_dict[trait['name']] = [float(ParticipantDto['placement'])]
```
Let's print out an example of what the trait_dict may look like after running the above code:
```python
print(trait_dict)
```
{  
'Set2_Assassin': [2.0, 2.0, 6.0, 4.0, 6.0, 5.0, 7.0, 6.0, 1.0],  
'Avatar': [6.0, 3.0],  
'Mountain': [6.0, 7.0, 2.0, 8.0, 6.0, 5.0],  
'Wind': [6.0],  
'Berserker': [7.0, 4.0, 1.0, 3.0],  
'Alchemist': [4.0, 4.0, 1.0],  
'Poison': [4.0, 4.0, 1.0],  
'Set2_Glacial': [4.0, 5.0, 3.0],  
'Druid': [2.0, 6.0],  
'Mage': [2.0, 8.0, 5.0],  
'Woodland': [2.0, 6.0],  
'Set2_Blademaster': [2.0],  
'Desert': [4.0, 8.0],  
'Summoner': [4.0]  
}

Lastly, we can turn the dictionary in to a Pandas Dataframe and find the averages and counts for each trait:
```python
import pandas
import matplotlib.pyplot
import numpy

# orient='index' is to change the dictionary keys to rows, one alternative to different sized column error
trait_df = pandas.DataFrame.from_dict(trait_dict, orient='index')
print(trait_df.head(2))

# create copy to make average and count columns without including the other column
trait_test = trait_df.copy()
trait_df['averages'] = trait_test.mean(axis=1)
trait_df['count'] = trait_test.count(axis=1, numeric_only=True)
print(trait_df[['averages', 'count']])

trait_df.plot.bar(y='averages', rot=35, bottom=1)
matplotlib.pyplot.show()
```
| |0|1|2|3|4|5|6|7|8|
|---|---|---|---|---|---|---|---|---|---|
|Set2_Assassin|2.0|2.0|6.0|4.0|6.0|5.0|7.0|6.0|1.0|
|Avatar|6.0|3.0|NaN|NaN|NaN|NaN|NaN|NaN|NaN|


| |averages|count|
|---|---|---|
|Set2_Assassin|4.333333|9|
|Avatar|4.500000|2|
|Mountain|5.666667|6|
|Wind|6.000000|1|
|Berserker|3.750000|4|
|Alchemist|3.000000|3|
|Poison|3.000000|3|
|Set2_Glacial|4.000000|3|
|Druid|4.000000|2|
|Mage|5.000000|3|
|Woodland|4.000000|2|
|Set2_Blademaster|2.000000|1|
|Desert|6.000000|2|
|Summoner|4.000000|1|
