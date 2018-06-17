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
