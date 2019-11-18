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
