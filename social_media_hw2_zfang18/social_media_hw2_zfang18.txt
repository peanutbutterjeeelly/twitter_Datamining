import twitter
import json
import urllib
import sys
import time
import networkx as nx
import matplotlib.pyplot as plt
from functools import partial
from sys import maxsize as maxint
from urllib.error import URLError
from http.client import BadStatusLine
from operator import itemgetter



#zfang18
#Zhixuan Fang
#337310566
# log in function
def oauth_login():
    CONSUMER_KEY = "PCRhHGIGTvGHK7Yam1ySLFe1S"
    CONSUMER_SECRET = "6vshANH6FaB3wQs7ic4dzkK6MkpvjFqELT71i7zj7NjOtlo89n"
    OAUTH_TOKEN = "2712612025-VzcrKdzDAQS3jixlIJ8jA2jEPIY52kHoLdSDFw1"
    OAUTH_TOKEN_SECRET = "vrFlHaE1xaEhQvF69pkhM4daaKlYqgEHDmXmG5NAYzKlM"
    auth = twitter.oauth.OAuth(OAUTH_TOKEN, OAUTH_TOKEN_SECRET,CONSUMER_KEY, CONSUMER_SECRET)
    twitter_api = twitter.Twitter(auth=auth)
    return twitter_api


def oauth_login_2():
    CONSUMER_KEY = 'YnZOXoz0kmn90o2WOZUmrdMZu'
    CONSUMER_SECRET = 'CICZqLaAjzMQfHUlLyUMgOk68Ji3qwlvPTAI5jOmBy8LKQKtZfT'
    OAUTH_TOKEN = 'AAAAAAAAAAAAAAAAAAAAAIP5IAEAAAAAwQFlegRhIxLijFq2CH6PUPRdkBQ%3D6c4VFro9NMxhucisu'
    OAUTH_TOKEN_SECRET = 'X1TxpJk6Gsg7R7aBsTwVgmbpoZP30WY5L'

    auth = twitter.oauth.OAuth(OAUTH_TOKEN, OAUTH_TOKEN_SECRET, CONSUMER_KEY, CONSUMER_SECRET)
    twitter_api = twitter.Twitter(auth=auth)
    return twitter_api


# Show some chosen information of the users
def show_user_message(twitter_api, screen_name=None, id=None):
    if(screen_name!=None):
        response = twitter_api.users.show(screen_name=screen_name)
    else:
        response = twitter_api.users.show(id=id)
    print("id: ",response["id"])
    print("name: ", response["name"])
    print("screen_name: ", response["screen_name"])
    print("followers_count: ", response["followers_count"])
    print("friends_count: ", response["friends_count"])


# Form the cookbook
# Make_twitter_request using the Exception
def make_twitter_request(twitter_api_func, max_errors=10, *args, **kw):
    def handle_twitter_http_error(e, wait_period=2, sleep_when_rate_limited=True):
        if wait_period > 3600:  # Seconds
            print('Too many retries. Quitting.', file=sys.stderr)
            raise e
        if e.e.code == 401:
            print('Encountered 401 Error (Not Authorized)', file=sys.stderr)
            return None
        elif e.e.code == 404:
            print('Encountered 404 Error (Not Found)', file=sys.stderr)
            return None
        elif e.e.code == 429:
            print('Encountered 429 Error (Rate Limit Exceeded)', file=sys.stderr)
            if sleep_when_rate_limited:
                print("Retrying in 15 minutes...ZzZ...", file=sys.stderr)
                sys.stderr.flush()
                time.sleep(60 * 15 + 5)
                print('...ZzZ...Awake now and trying again.', file=sys.stderr)
                return 2
            else:
                raise e  # Caller must handle the rate limiting issue
        elif e.e.code in (500, 502, 503, 504):
            print('Encountered {0} Error. Retrying in {1} seconds'.format(e.e.code, wait_period), file=sys.stderr)
            time.sleep(wait_period)
            wait_period *= 1.5
            return wait_period
        else:
            raise e
    # End of nested helper function
    wait_period = 2
    error_count = 0

    while True:
        try:
            return twitter_api_func(*args, **kw)
        except twitter.api.TwitterHTTPError as e:
            error_count = 0
            wait_period = handle_twitter_http_error(e, wait_period)
            if wait_period is None:
                return
        except URLError as e:
            error_count += 1
            time.sleep(wait_period)
            wait_period *= 1.5
            print("URLError encountered. Continuing.", file=sys.stderr)
            if error_count > max_errors:
                print("Too many consecutive errors...bailing out.", file=sys.stderr)
                raise
        except BadStatusLine as e:
            error_count += 1
            time.sleep(wait_period)
            wait_period *= 1.5
            print("BadStatusLine encountered. Continuing.", file=sys.stderr)
            if error_count > max_errors:
                print("Too many consecutive errors...bailing out.", file=sys.stderr)
                raise



# Form the cookbook
# Get the friends and followers ids
def get_friends_followers_ids(twitter_api, screen_name=None, user_id=None,friends_limit=maxint, followers_limit=maxint):
    assert (screen_name != None) != (user_id != None), "Must have screen_name or user_id, but not both"
    get_friends_ids = partial(make_twitter_request, twitter_api.friends.ids, count=5000)  # TODO
    get_followers_ids = partial(make_twitter_request, twitter_api.followers.ids, count=5000)
    friends_ids, followers_ids = [], []
    for twitter_api_func, limit, ids, label in \
    [
        [get_friends_ids, friends_limit, friends_ids, "friends"],
        [get_followers_ids, followers_limit, followers_ids, "followers"]
    ]:
        if limit == 0:
            continue
        cursor = -1
        while cursor != 0:
            if screen_name:
                response = twitter_api_func(screen_name=screen_name, cursor=cursor)
            else:  # user_id
                response = twitter_api_func(user_id=user_id, cursor=cursor)
            if response is not None:
                ids += response['ids']
                cursor = response['next_cursor']
            print('Fetched {0} total {1} ids for {2}'.format(len(ids), label, (user_id or screen_name)),file=sys.stderr)
            if len(ids) >= limit or response is None:
                break
    return friends_ids[:friends_limit], followers_ids[:followers_limit]


# Form the cookbook
# Get the information of the chosen friend
def get_user_profile(twitter_api, screen_names=None, user_ids=None):
    assert (screen_names != None) != (user_ids != None),"Must have one of them"
    items_to_info = {}
    items = screen_names or user_ids
    while len(items) > 0:
        # Process 100 items at a time per the API specifications for /users/lookup.
        items_str = ','.join([str(item) for item in items[:100]])
        #print("items_str:",items_str)
        items = items[100:]
        #print("items_after:",items)
        if screen_names:
            response = make_twitter_request(twitter_api.users.lookup, screen_name=items_str)
        else:
            #print("the input is users ids")
            response = make_twitter_request(twitter_api.users.lookup, user_id=items_str)
        for user_info in response:
            if screen_names:
                items_to_info[user_info['screen_name']] = user_info
            else:
                items_to_info[user_info['id']] = user_info
    return items_to_info


def finding_reciprocal_friends(friends_ids,followers_ids):
    reciprocal_friends = set(friends_ids) & set(followers_ids)
    print("Total reciprocal friends:", len(reciprocal_friends), file=sys.stderr)
    return reciprocal_friends



def popular_friends(twitter_api,reciprocal_friends):
    response=get_user_profile(twitter_api,user_ids=reciprocal_friends) # find the users information of the reciprocal friends
    #print(json.dumps(response,indent=1))
    dict={}
    for id in response:
        followers_count=response[id]['followers_count']   # need the followers_count
        dict[id]=followers_count
        #print("id:",id,"     followers_count:",response[id]['followers_count'])
    dict_sorted=sorted(dict.items(),key=itemgetter(1),reverse=True)  # This can get the tuple which has both the number of followers and id
    list_sorted=sorted(dict,key=dict.get,reverse=True)
    print("Popular top 5 friends(id,followers_count):", dict_sorted[0:5])#, file=sys.stderr
    return list_sorted[0:5]


def crawler(twitter_api,point_screen_name,Least_nodes):
    # step 2 of starting point
    friends_ids, followers_ids = get_friends_followers_ids(twitter_api, screen_name=point_screen_name, friends_limit=5000, followers_limit=5000) # get the two information of the starting point
    print("The starting point's friends ids:", friends_ids)
    print("The starting point's friends ids:", followers_ids)
    starting_point_id = twitter_api.users.show(screen_name=point_screen_name)["id"]   # get the id of the staring point
    # step 3 of starting point
    reciprocal_friends = list(finding_reciprocal_friends(friends_ids, followers_ids))
    print("The starting point's reciprocal friends:",reciprocal_friends)
    # step 4 of starting point
    popular_friends_top_5=popular_friends(twitter_api, reciprocal_friends)
    print("The starting point's top 5 popular friends:",popular_friends_top_5)

    nodes = []  # the list of the all nodes
    edges = []  # the list of the tuples of the all edges
    next_queue = popular_friends_top_5
    nodes.append(starting_point_id)
    for i in popular_friends_top_5:
        nodes.append(i)
        edges.append((starting_point_id,i))

    while True:   #TODO
        (queue, next_queue) = (next_queue, [])
        for id in queue:
            # step 2
            friends_ids, followers_ids = get_friends_followers_ids(twitter_api, user_id=id,friends_limit=5000, followers_limit=5000)
            # step 3
            reciprocal_friends = list(finding_reciprocal_friends(friends_ids, followers_ids))
            # step 4
            popular_friends_top_5 = popular_friends(twitter_api, reciprocal_friends)
            if popular_friends_top_5:
                for i in popular_friends_top_5:
                    if (i in nodes or i in next_queue):  # if in ids or next_queue
                        edges.append((id, i))  # if the i in ids or i in next_queue, it means this friend has relation with the node already have
                        continue
                    else:
                        next_queue.append(i) # add the i in the next queue to find another step
                        edges.append((id,i))  # add the tuple in the list edges
                        nodes.append(i)  # add the id in the nodes
            if(len(nodes)>Least_nodes):  # If nodes are more than 100, return the result
                return nodes, edges



def Graph(nodes,edges):
    G = nx.Graph()  # or DiGraph(), MultiGraph(), MultiDiGraph() // see next slide
    G.add_nodes_from(nodes)
    G.add_edges_from(edges)
    print("Nodes：{}".format(G.nodes()))
    print("Edges：{}".format(G.edges()))
    print("Nodes number：{}".format(G.number_of_nodes()))
    print("Edges number：{}".format(G.number_of_edges()))

    # this is to get the graph with the labels
    nx.draw(G,node_size=40,with_labels=nodes)
    plt.show()

    # this is to get the graph with the labels
    nx.draw(G,node_size=40)
    plt.show()
    print("Nodes number：{}".format(G.number_of_nodes()))
    print("Edges number：{}".format(G.number_of_edges()))
    print("Diameter:",nx.diameter(G))  # show the Diameter of the graph
    print("Average distance:",nx.average_shortest_path_length(G)) # show the Average distance of the graph




def __main__():
    # login in
    twitter_api = oauth_login()

    # step 1
    starting_point = 'daydreamingiwz'#'edmundyu1001'
    print("starting_point:",starting_point)
    Least_nodes=100
    nodes,edges=crawler(twitter_api,starting_point,Least_nodes)
    Graph(nodes,edges)

if __name__=="__main__":
    __main__()


















