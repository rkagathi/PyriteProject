# PyriteProject

import pandas as pd
import tweepy
import time
import json
# from replies import *
from datetime import *
from pprint import pprint
from config import consumer_key, consumer_secret, access_token, access_token_secret
# import geoplot
import folium
# Import and Initialize Sentiment Analyzer
from vaderSentiment.vaderSentiment import SentimentIntensityAnalyzer
analyzer = SentimentIntensityAnalyzer()
# USA_COORDINATES = (41.8781,-87.6298)

# Setup Tweepy API Authentication
auth = tweepy.OAuthHandler(consumer_key, consumer_secret)
auth.set_access_token(access_token, access_token_secret)
api = tweepy.API(auth, parser=tweepy.parsers.JSONParser())

#import csv
file = "2016ElectionDataByState.csv"
df = pd.read_csv(file)
del df['Unnamed: 3']

temp = (df['Clinton'] + df['Trump'])
df['Clinton'] = df['Clinton'].str.replace(",","").astype(int)
df['Clinton'] = pd.to_numeric(df["Clinton"])
df['Trump'] = df['Trump'].str.replace(",","").astype(int)
df['Trump'] = pd.to_numeric(df["Trump"])
df['percent_clinton'] = df['Clinton']/(df['Clinton'] + df['Trump'])
df['percent_trump'] = df['Trump']/(df['Clinton'] + df['Trump'])
df['delta_trump'] = df['percent_trump'] - df['percent_clinton']
df.to_csv("Output/2016Voting.csv", index=False, header=True)

#import State lat/lng csv
llfile = "statelatlong.csv"
state_ll_df = pd.read_csv(llfile)
state_ll_df['location'] = state_ll_df['City']

states = ["AL", "AK", "AZ", "AR", "CA", "CO", "CT", "DC", "DE", "FL", "GA", 
          "HI", "ID", "IL", "IN", "IA", "KS", "KY", "LA", "ME", "MD", 
          "MA", "MI", "MN", "MS", "MO", "MT", "NE", "NV", "NH", "NJ", 
          "NM", "NY", "NC", "ND", "OH", "OK", "OR", "PA", "RI", "SC", 
          "SD", "TN", "TX", "UT", "VT", "VA", "WA", "WV", "WI", "WY"]
          
# Target Term
target_user = "@realDonaldTrump"

datetime_object = datetime.strptime('Apr 27 2018 7:15AM', '%b %d %Y %I:%M%p')
datetime_object
#     if datetime.strptime(tweet['created_at'],'%a %b %d %H:%M:%S +0000 %Y') <= datetime_object:

# Get all tweets from target user
tweet_list = []
page = 1
found = False
while found == False:
    public_tweets = api.user_timeline(target_user, page=page)
    original_tweet = 0
    # Loop through all tweets
    for i, tweet in enumerate(public_tweets):
        if datetime.strptime(tweet['created_at'],'%a %b %d %H:%M:%S +0000 %Y') <= datetime_object:
            found = True
            tweet_list.append(tweet)
            replies_list = []    
            orginal_tweet = tweet.get('id')
            pprint(datetime.strptime(tweet['created_at'],'%a %b %d %H:%M:%S +0000 %Y'))

            replies = api.search(q='to:@realDonaldTrump',since_id=tweet.get('id'),result_type='oldest', count=100,geocode='41.8781,-87.6298,2000mi')
            while len(replies.get('statuses')) > 0 and replies.get('statuses')[0].get('id') > original_tweet:
                for reply in replies["statuses"]:
        #             if reply.get('coordinates') != None:
                    replies_list.append(reply) 
                replies = api.search(q='to:@realDonaldTrump',since_id=reply.get('id'), result_type='oldest', count=100,geocode='41.8781,-87.6298,2000mi')
                original_tweet = reply.get('id')  
                if (len(replies_list) > 10000):
                    break
            break
        else:
            page+=1
            found = False
            
len(tweet_list), len(replies_list)

pprint(tweet_list[0].get('id')), pprint(tweet_list[0].get('created_at')), pprint(tweet_list[0].get('text'))

# for reply in replies_list:# for re 
#     pprint(reply.get('id'))
#     pprint(reply.get('created_at'))
#     pprint(reply.get('text'))
#     pprint(reply.get('coordinates'))
#     break
    
replies_df = pd.DataFrame(replies_list)

replies_list[0].get('user').get('location')
replies_list[0]

# [x['user'].get('location', None) for x in replies_list]

flat_replies = []
# found_State = False
for reply in replies_list:
    if len(reply.get('user').get('location')) > 0:
        found = False
        for state in states:
            if (reply.get('user').get('location').find(state) > 0):
                found = True
                break
        # Run Vader Analysis on each tweet
        if found:
            results = analyzer.polarity_scores(reply.get('text'))
            reply_dict = {'id': reply.get('id'),
                        'text' : reply.get('text'),
                        'location': state,
                        'tweet_dt': reply.get('created_at'),  
                        'compound': results["compound"],
                        'pos': results["pos"],
                        'neu': results["neu"],
                        'neg': results["neg"]}
            flat_replies.append(reply_dict)
            
replies_df = pd.DataFrame(flat_replies)
tweetsByState_df = replies_df.groupby('location').count().reset_index()

replies_df

sorted_df = replies_df.groupby("location")["compound"].mean().reset_index()
# sorted_df

sorted_df.sort_values('compound').reset_index(drop=True)

# import os
# os.listdir()

# sorted_df['pos_compound'] = sorted_df['compound'] + 1

sorted_df

#sorted_df.columns

# print(map.choropleth.__doc__)

# with open("state.geojson.json") as fin:
#     data = json.load(fin)

# data.keys()

# data.get('features')[0]['properties']

map = folium.Map(location=USA_COORDINATES, zoom_start=4)
# map.geojson(geo_path='state.geo.json')
# map
# folium.GeoJson(data='state.geojson.json',
#              ).add_to(map)
# display(map)
map.choropleth(geo_data='state.geojson.json', 
                   data=sorted_df, 
                  columns=['location', 'compound'],
                  key_on='feature.properties.STUSPS10',
                  fill_color='BuPu')

display(map)

df.head()

map_votes = folium.Map(location=USA_COORDINATES, zoom_start=4)
# map.geojson(geo_path='state.geo.json')
# map
# folium.GeoJson(data='state.geojson.json',
#              ).add_to(map)
# display(map)
map_votes.choropleth(geo_data='state.geojson.json', 
                   data=df, 
                  columns=['State', 'delta_trump'],
                  key_on='feature.properties.NAME10',
                  fill_color='BuPu')

display(map_votes)
# map_votes.savefig("output/2016Voting.png")

map = folium.Map(location=USA_COORDINATES, zoom_start=4)

map.choropleth(geo_data='state.geojson.json', 
                   data=sorted_df, 
                   threshold_scale=[-1, -0.5, -0, 0.5, 1],
                  columns=['location', 'compound'],
                  key_on='feature.properties.statename',
                  fill_color='BuPu', fill_opacity=0.7, line_opacity=0.5,)

display(map)

# map

import sys
sys.stdout.encoding

SENTIMENT ANALYSIS

import pandas as pd
import tweepy
import time
import json
# from replies import *
from pprint import pprint
# from config import consumer_key, consumer_secret, access_token, access_token_secret
# Import and Initialize Sentiment Analyzer
from vaderSentiment.vaderSentiment import SentimentIntensityAnalyzer
analyzer = SentimentIntensityAnalyzer()
consumer_key = f'oypRxofLOoARuFD61NQ8gZlhK'
consumer_secret = f'F4qJiwciKfhmiPrcWJTJ0dLhx94PaBmRhSXbAoZqUH8MxgQUeN'
access_token = f'998710477411835904-eQYIo9recCc1JzqaaHjiL03bzWkR1Vx'
access_token_secret = f'7JZmaIXCDYMOauLJhcSm6S5U5IbxkMOUkCXncNdpEuvVu'

# Setup Tweepy API Authentication
auth = tweepy.OAuthHandler(consumer_key, consumer_secret)
auth.set_access_token(access_token, access_token_secret)
api = tweepy.API(auth, parser=tweepy.parsers.JSONParser())

#  state_fn = ["Alabama", "Alaska", "Arizona", "Arkansas", "California", "Colorado", "Connecticut", "DC", "Delaware", "Florida", "Georgia", 
#            "Hawaii", "Idaho", "Illinois", "Indiana", "Iowa", "Kansas", "Kentucky", "Louisiana", "Maine", "Maryland", 
#            "Massachusetts", "Michigan", "Minnesota", "Mississippi", "Missouri", "Montana", "Nebraska", "Nevada", "New Hampshire", "New Jersey", 
#            "New Mexico", "New York", "North Carolina", "North Dakota", "Ohio", "Oklahoma", "Oregon", "Pennsylvania", "Rhode Island", "South Carolina", 
#            "South Dakota", "Tennessee", "Texas", "Utah", "Vermont", "Virginia", "Washingtion", "West Virginia", "Wyoming"]

states = ["AL", "AK", "AZ", "AR", "CA", "CO", "CT", "DC", "DE", "FL", "GA", 
          "HI", "ID", "IL", "IN", "IA", "KS", "KY", "LA", "ME", "MD", 
          "MA", "MI", "MN", "MS", "MO", "MT", "NE", "NV", "NH", "NJ", 
          "NM", "NY", "NC", "ND", "OH", "OK", "OR", "PA", "RI", "SC", 
          "SD", "TN", "TX", "UT", "VT", "VA", "WA", "WV", "WI", "WY"]
          
# Target Term
target_user = "@realDonaldTrump"

# Get all tweets from target user
tweet_list = []
public_tweets = api.user_timeline(target_user, page=1)
original_tweet = 0
# Loop through all tweets
for i, tweet in enumerate(public_tweets):
    tweet_list.append(tweet)
    replies_list = []
    orginal_tweet = tweet.get('id')

    replies = api.search(q='to:@realDonaldTrump',since_id=tweet.get('id'),count=100,geocode='41.8781,-87.6298,2000mi')
    while len(replies.get('statuses')) > 0 and replies.get('statuses')[0].get('id') > original_tweet:
        for reply in replies["statuses"]:
#             if reply.get('coordinates') != None:
            replies_list.append(reply) 
        replies = api.search(q='to:@realDonaldTrump',since_id=reply.get('id'), count=100,geocode='41.8781,-87.6298,2000mi')
        original_tweet = reply.get('id')
        if (len(replies_list) > 6000):
            break
    break
    
    # # Target Term
# target_user = "@realDonaldTrump"

# # Get all tweets from target user
# tweet_list = []
# public_tweets = api.user_timeline(target_user, page=1)
# original_tweet = 0
# # Loop through all tweets
# for i, tweet in enumerate(public_tweets):
#     tweet_list.append(tweet)
#     replies_list = []
#     orginal_tweet = tweet.get('id')

#     replies = api.search(q='to:@realDonaldTrump',since_id=tweet.get('id'),count=100,geocode='41.8781,-87.6298,2000mi')
#     while len(replies.get('statuses')) > 0 and replies.get('statuses')[0].get('id') > original_tweet:
#         for reply in replies["statuses"]:
# #             if reply.get('coordinates') != None:
#             replies_list.append(reply) 
#         replies = api.search(q='to:@realDonaldTrump',since_id=reply.get('id'), count=100,geocode='41.8781,-87.6298,2000mi')
#         original_tweet = reply.get('id')
#         if (len(replies_list) > 6000):
#             break
#     break
len(tweet_list), len(replies_list)

flat_replies = []
# found_State = False
for reply in replies_list:
   if len(reply.get('user').get('location')) > 0:
       found = False
       for state in states:
           if (reply.get('user').get('location').find(state) > 0):
               found = True
               break
       # Run Vader Analysis on each tweet
       if found:
           results = analyzer.polarity_scores(reply.get('text'))
           reply_dict = {'id': reply.get('id'),
                       'text' : reply.get('text'),
                       'location': state,
                       'compound': results["compound"],
                       'pos': results["pos"],
                       'neu': results["neu"],
                       'neg': results["neg"]}
           flat_replies.append(reply_dict)
        
pprint(tweet_list[0].get('id')), pprint(tweet_list[0].get('created_at')), pprint(tweet_list[0].get('text'))

replies_df = pd.DataFrame(replies_list)

replies_list[0].get('user').get('location')
replies_list[0]


replies_dfreplies_d  = pd.DataFrame(flat_replies)
replies_df

None

state_compound_fakenews = replies_df.groupby("location")["compound"].mean().reset_index()

state_compound_fakenews

state_compound_fakenews.to_csv("state_compound_fakenews.csv", encoding="utf-8", index=False)

state_pos_fakenews = replies_df.groupby("location")["pos"].mean().reset_index()

state_pos_fakenews

state_pos_fakenews.to_csv("state_pos_fakenews", encoding="utf-8", index=False)

state_neg_fakenews = replies_df.groupby("location")["neg"].mean().reset_index()

state_neg_fakenews.to_csv("state_neg_fakenews.csv", encoding="utf-8", index=False)

replies_df.sort_values(by=['location'])

None
 
import numpy as np
import matplotlib.pyplot as plt

state_bar = state_compound_df['location']
state_compound = state_compound_df['compound']
x_axis = np.arange(0, len(state_bar), 1)
plt.bar(x_axis, state_compound, align="center")
# Set x axis tick locations
tick_locations = [value+.1 for value in x_axis]
plt.xticks(tick_locations, state_bar)



# Set the limits of the x axis
plt.xlim(-0.5, len(x_axis))

# Set the limits of the y axis
plt.ylim(-1, 1)



import folium

USA_COORDINATES = (41.8781,-87.6298)

state_geo = r'state.geo.json'

map = folium.Map(location=USA_COORDINATES, zoom_start=4)
# map.geojson(geo_path='state.geo.json')
# map
# folium.GeoJson(data='state.geojson.json',
#              ).add_to(map)
# display(map)
map.choropleth(geo_data='state.geo.json',
                  data=state_compound_df,
                 columns=['location', 'compound'],
                 key_on='feature.properties.STUSPS10',
                 fill_color='BuPu')

map

None

tweet_2_06_11_18 = 'A year ago the pundits & talking heads, people that couldn’t do the job before, were begging for conciliation and peace - “please meet, don’t go to war.” Now that we meet and have a great relationship with Kim Jong Un, the same haters shout out, “you shouldn’t meet, do not meet!”'
results = analyzer.polarity_scores(tweet_2_06_11_18)
pos = results["pos"]
neg = results["neg"]
neu = results["neu"]
comp = results["compound"]

tweet_2_06_11_18_df =[pos, neg, neu, comp]

tweet_2_06_11_18_df

tweet_sentiment2 = pd.read_csv('cleaned_tweet2_sentiment.csv')

tweet_sentiment2

state_bar = tweet_sentiment2['State']
state_compound = tweet_sentiment2['compound']
x_axis = np.arange(0, len(state_bar), 1)
plt.bar(x_axis, state_compound, align="center")
# Set x axis tick locations
tick_locations = [value+.1 for value in x_axis]
plt.xticks(tick_locations, state_bar)



# Set the limits of the x axis
plt.xlim(-0.5, len(x_axis))

# Set the limits of the y axis
plt.ylim(-1, 1)

from scipy.stats import linregress

# Set data
x_axis = tweet_sentiment2['Trump Proportion']
y_axis= tweet_sentiment2['pos']

# Set line
(slope, intercept, _, _, _) = linregress(x_axis, y_axis)
fit = slope * x_axis + intercept

# Plot data
fig, ax = plt.subplots()

fig.suptitle("Tweet #3 Positive State Sentiment by 2016 Trump State Vote Share", fontsize=16, fontweight="bold")

ax.set_xlim(0, 1)
ax.set_ylim(0, 1)

ax.set_xlabel("Trump Vote Proportion by State")
ax.set_ylabel("Positive State Sentiment")

ax.plot(x_axis, y_axis, linewidth=0, marker='o')
ax.plot(x_axis, fit, 'b--')

plt.grid()

plt.show()

# Set data
x_axis = tweet_sentiment2['Trump Proportion']
y_axis= tweet_sentiment2['neg']

# Set line
(slope, intercept, _, _, _) = linregress(x_axis, y_axis)
fit = slope * x_axis + intercept

# Plot data
fig, ax = plt.subplots()

fig.suptitle("Tweet #3 Negative State Sentiment by 2016 Trump State Vote Share", fontsize=16, fontweight="bold")

ax.set_xlim(0, 1)
ax.set_ylim(0, 1)

ax.set_xlabel("Trump Vote Proportion by State")
ax.set_ylabel("Negative State Sentiment")

ax.plot(x_axis, y_axis, linewidth=0, marker='o')
ax.plot(x_axis, fit, 'b--')

plt.grid()

plt.show()

tweet1_sentiment= pd.read_csv('state_sentiment_df_612_1.csv')

tweet1_sentiment

# Set data
x_axis = tweet1_sentiment['Trump Proportion of Vote']
y_axis= tweet1_sentiment['pos']

# Set line
(slope, intercept, _, _, _) = linregress(x_axis, y_axis)
fit = slope * x_axis + intercept

# Plot data
fig, ax = plt.subplots()

fig.suptitle("Tweet #2 Positive State Sentiment by 2016 Trump State Vote Share", fontsize=16, fontweight="bold")

ax.set_xlim(0, 1)
ax.set_ylim(0, 1)

ax.set_xlabel("Trump Vote Proportion by State")
ax.set_ylabel("Positive State Sentiment")

ax.plot(x_axis, y_axis, linewidth=0, marker='o')
ax.plot(x_axis, fit, 'b--')

plt.grid()

plt.show()

# Set data
x_axis = tweet1_sentiment['Trump Proportion of Vote']
y_axis= tweet1_sentiment['neg']

# Set line
(slope, intercept, _, _, _) = linregress(x_axis, y_axis)
fit = slope * x_axis + intercept

# Plot data
fig, ax = plt.subplots()

fig.suptitle("Tweet #2 Negative State Sentiment by 2016 Trump State Vote Share", fontsize=16, fontweight="bold")

ax.set_xlim(0, 1)
ax.set_ylim(0, 1)

ax.set_xlabel("Trump Vote Proportion by State")
ax.set_ylabel("Negative State Sentiment")

ax.plot(x_axis, y_axis, linewidth=0, marker='o')
ax.plot(x_axis, fit, 'b--')

plt.grid()

plt.show()

state_bar = tweet1_sentiment['location']
state_compound = tweet1_sentiment['compound']
x_axis = np.arange(0, len(state_bar), 1)
plt.bar(x_axis, state_compound, align="center")
# Set x axis tick locations
tick_locations = [value+.1 for value in x_axis]
plt.xticks(tick_locations, state_bar)



# Set the limits of the x axis
plt.xlim(-0.5, len(x_axis))

# Set the limits of the y axis
plt.ylim(-1, 1)

otweet_sentiment = pd.read_csv('0tweet_sentiment.csv')
 
otweet_sentiment

# Set data
x_axis = otweet_sentiment['Trump Vote']
y_axis= otweet_sentiment['neg']

# Set line
(slope, intercept, _, _, _) = linregress(x_axis, y_axis)
fit = slope * x_axis + intercept

# Plot data
fig, ax = plt.subplots()

fig.suptitle("Tweet #1 Negative State Sentiment by 2016 Trump State Vote Share", fontsize=16, fontweight="bold")

ax.set_xlim(0, 1)
ax.set_ylim(0, 1)

ax.set_xlabel("Trump Vote Proportion by State")
ax.set_ylabel("Negative State Sentiment")

ax.plot(x_axis, y_axis, linewidth=0, marker='o')
ax.plot(x_axis, fit, 'b--')

plt.grid()

plt.show()

# Set data
x_axis = otweet_sentiment['Trump Vote']
y_axis= otweet_sentiment['pos']

# Set line
(slope, intercept, _, _, _) = linregress(x_axis, y_axis)
fit = slope * x_axis + intercept

# Plot data
fig, ax = plt.subplots()

fig.suptitle("Tweet #1 Positive State Sentiment by 2016 Trump State Vote Share", fontsize=16, fontweight="bold")

ax.set_xlim(0, 1)
ax.set_ylim(0, 1)

ax.set_xlabel("Trump Vote Proportion by State")
ax.set_ylabel("Positive State Sentiment")

ax.plot(x_axis, y_axis, linewidth=0, marker='o')
ax.plot(x_axis, fit, 'b--')

plt.grid()

plt.show()

state_bar = otweet_sentiment['State']
state_compound = otweet_sentiment['compound']
x_axis = np.arange(0, len(state_bar), 1)
plt.bar(x_axis, state_compound, align="center")
# Set x axis tick locations
tick_locations = [value+.1 for value in x_axis]
plt.xticks(tick_locations, state_bar)



# Set the limits of the x axis
plt.xlim(-0.5, len(x_axis))

# Set the limits of the y axis
plt.ylim(-1, 1)

tweet_2_06_12_18 = f'Its time for another #MAGA rally. Join me in Duluth, Minnesota on Wednesday, June 20th at 6:30pm! Tickets➜http://donaldjtrump.com/rallies/duluth-mn-june-2018 …'
results = analyzer.polarity_scores(tweet_2_06_12_18)
pos = results["pos"]
neg = results["neg"]
neu = results["neu"]
comp = results["compound"]

comp

pos

neg

neu


state_compound_fakenewsstate_com

tweet_fakenews = pd.read_csv('state_fakenews_df.csv')

tweet_fakenews

None

# Set data
x_axis = tweet_fakenews['TrumpVote']
y_axis= tweet_fakenews['pos']

# Set line
(slope, intercept, _, _, _) = linregress(x_axis, y_axis)
fit = slope * x_axis + intercept

# Plot data
fig, ax = plt.subplots()

fig.suptitle("Tweet #4 Positive State Sentiment by 2016 Trump State Vote Share", fontsize=16, fontweight="bold")

ax.set_xlim(0, 1)
ax.set_ylim(0, 1)

ax.set_xlabel("Trump Vote Proportion by State")
ax.set_ylabel("Positive State Sentiment")

ax.plot(x_axis, y_axis, linewidth=0, marker='o')
ax.plot(x_axis, fit, 'b--')

plt.grid()

plt.show()

# Set data
x_axis = tweet_fakenews['TrumpVote']
y_axis= tweet_fakenews['neg']

# Set line
(slope, intercept, _, _, _) = linregress(x_axis, y_axis)
fit = slope * x_axis + intercept

# Plot data
fig, ax = plt.subplots()

fig.suptitle("Tweet #4 Negative State Sentiment by 2016 Trump State Vote Share", fontsize=16, fontweight="bold")

ax.set_xlim(0, 1)
ax.set_ylim(0, 1)

ax.set_xlabel("Trump Vote Proportion by State")
ax.set_ylabel("Negative State Sentiment")

ax.plot(x_axis, y_axis, linewidth=0, marker='o')
ax.plot(x_axis, fit, 'b--')

plt.grid()

plt.show()

# Set data
x_axis = tweet_fakenews['TrumpVote']
y_axis= tweet_fakenews['compound']

# Set line
(slope, intercept, _, _, _) = linregress(x_axis, y_axis)
fit = slope * x_axis + intercept

# Plot data
fig, ax = plt.subplots()

fig.suptitle("Tweet #4 Negative State Sentiment by 2016 Trump State Vote Share", fontsize=16, fontweight="bold")

ax.set_xlim(0, 1)
ax.set_ylim(0, 1)

ax.set_xlabel("Trump Vote Proportion by State")
ax.set_ylabel("Negative State Sentiment")

ax.plot(x_axis, y_axis, linewidth=0, marker='o')
ax.plot(x_axis, fit, 'b--')

plt.grid()

plt.show()

WRITEUP
In our project, we elected to analyze responses to President Donald Trump’s tweets using VADER Sentiment Analysis, a sentiment analysis tool focused towards social media use. Our motivation behind this decision was a culmination of our newfound knowledge regarding working with tweepy and Twitter’s API as well as an interest in our current political climate and the President’s use of social media. Trump’s use of social media (Twitter, specifically) is  fcomparatively raw, less polished and more emotional when compared to Obama, the only other president who was has used Twitter as a legitimate platform. The content of Trump’s tweets have had large and varied impacts on a range of real-world matters across the country from court cases in both the US Court of Appeals and the US District Court to the stock market. This prompted us to delve into what affects Trump’s tweets had on other Twitter users. We were also interested in discovering whether any correlation existed between sentiments in response to Trump and geographic Presidential Election results.
In order to narrow our focus, we concentrated on the following questions and subsequent hypotheses. 
1.	Do users who live in states that voted for the President in the 2016 election express more positive sentiments in reply to the @realDonaldTrump relative to users who do not?
2.	Does the sentiment expressed within the President’s twees(s) affect/directly correlate with user sentiment? 
We were ultimately unable to answer this question due to limitations within our data. Our predictive variables had too much variance (we need predictive variables with no variance in order to accurately measure and compare sentiments in a statistically significant manner or more advanced sentiment analysis tools to use in conjunction with VADER)
3.	How Does the average sentiment vary between states? Does the proportion of the 2016 vote share by state predict current difference in the sentiment expressed in reply to the President? 
Our null hypotheses were:
1.	Users who live in states that voted for Trump will express more positive sentiment whereas voters who live in states that voted for Clinton will express more negative sentiment.
2.	The sentiment of the President’s tweet will directly affect the sentiment of the users’ replies.
3.	The proportion of the 2016 vote share by state will predict the extremity of reply sentiment within states and reflect vote share differences. 
We gathered our most pertinent data from Twitter, using Twitter’s API to harvest all the replies to the POTUS’s most recent tweet. We spent approximately 90% of our time on what we needed to write to harvest this data correctly and then clean, sort and group that data a meaningful manner for analysis. To do this, we used the ‘since_id’ index and Trumps Twitter handle to gather responses to his tweets. After creating data frames (for each tweet we gathered responses from), we filtered out any tweets that did not provide a user location that was state specific so we could compare these tweets to the votes. Additionally, we gathered election data that gave a breakdown on the number of individual votes each major candidate (Clinton and Trump) received from each state. Using folium, we created a heatmap showing the percentage difference in votes for Trump. Trump had a higher voter share in the darker states. This created a baseline for us to use in our sentiment analysis. 
Our findings were not statistically significant. Due to time constraints and rate limits, we gathered data for three tweets. The first read, “Our Great Larry Kudlow, who has been working so hard on trade and the economy, has just suffered a heart attack He is now in the Walter Reed Medical Center.” VADER gave us a moderately negative compound score for this tweet. We gathered responses from 24 states and passed those tweets through the VADER analysis tool as well. We created another heatmap with these sentiments as an indicator and saw little correspondence with our baseline or the election data. The second tweet we analyzed read, “It’s time for another #MAGA rally. Join me in Duluth, Minnesota on Wednesday, June 20th at 6:30pm!” VADER gave us an overall fairly neutral score for this tweet. We gathered responses for this tweet, filtered them based on whether the user provided their location, and plotted these sentiments on a heatmap. We were surprised to find consistency between the sentiments in response to the above mentioned tweet and the baseline heatmap. The last tweet we analyzed read, “A year ago the pundits & talking head, people that couldn’t do the job before, were begging for conciliation and peace – “please meet, don’t go to war.” Now that we meet and have a great relationship with Kim Jong Un, the same haters shout out, ‘you shouldn’t meet, do not meet!’” After passing this tweet through VADER, a limitation in the tool was exposed – this tweet received a compound score of .141, a positive score of .141, a negative score of just .057, and a neutral score of .802. We passed responses to this tweet through VADER and mapped responses on a heatmap. Again, we did not see any correlation to the election results. 
Due to these results, we failed to reject each null hypothesis as our data did not show any statistically significant evidence to support them. Our belief is that this is a result of many factors. Public sentiment shifts constantly over time. Secondly, traditional quantitative data, such as state vote counts and proportions, may fail to adequately predict public sentiment in real time. Additionally, user reply sentiment does not appear to vary systematically from state to state. Our research only shows responses from the most recent tweets, Twitter users may not be the same people who participated in the 2016 Presidential Election, our time scale was limited, and our analysis only examined user replies and excluded any images, gifs, and content. Knowing our limitations and constraints, in the future we believe that in collecting replies from Trump’s tweets and then categorizing them based on the subject of the initial tweet may give us more accurate results when running any sort of sentiment analysis. We would also like to move analysis beyond the scope of individual lexical items and code for context and statement level responses. Of course, we find it would be beneficial to extend time course series and expand the sample size of responses analyzed. 
