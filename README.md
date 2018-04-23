
# Analyzing iMessage Conversations

## Notebook Sections: 
---
1. Load
2. Investigate
3. Extract
4. Transform
5. Visualize


```python
import sqlite3
import pandas as pd
import numpy as np
import string
import re

from plotly.offline import download_plotlyjs, init_notebook_mode, iplot
import plotly.plotly as py
import plotly.graph_objs as go
init_notebook_mode(connected=True)
```


<script>requirejs.config({paths: { 'plotly': ['https://cdn.plot.ly/plotly-latest.min']},});if(!window.Plotly) {{require(['plotly'],function(plotly) {window.Plotly=plotly;});}}</script>


## Load chat data from sqlite
----------------------------


```python
# helper functions for connecting to SQLite3 with python
def connect(sqlite_file):
    """ Make connection to an SQLite database file """
    conn = sqlite3.connect(sqlite_file)
    c = conn.cursor()
    return conn, c

def close(conn):
    """ Commit changes and close connection to the database """
    conn.close()
    
def table_col_info(cursor, table_name, print_out=False):
    """ Returns a list of tuples with column informations:
    (id, name, type, notnull, default_value, primary_key)"""
    cursor.execute('PRAGMA TABLE_INFO({})'.format(table_name))
    info = cursor.fetchall()

    if print_out:
        print("\nColumn Info:\nID, Name, Type, NotNull, DefaultVal, PrimaryKey")
        for col in info:
            print(col)

def get_col_names(sql_table):
    '''return a list of column names from sql query table'''
    return list(map(lambda x: x[0], table.description))
```

## Investigate Database
----------


```python
# sqlite path to chat.db
sqlite_path = '/Users/adamstueckrath/Library/Messages/chat.db'

# connect to db and get cursor to execute sql commands
sqlite_db, sql_command = connect(sqlite_path)
```


```python
# list of available tables in database, chat.db
table_name_query = '''
                   SELECT name 
                   FROM sqlite_master 
                   WHERE type='table';
                   '''
table_names = sql_command.execute(table_name_query)
table_list = [table[0] for table in table_names]
print(table_list)
```

    ['_SqliteDatabaseProperties', 'deleted_messages', 'sqlite_sequence', 'chat_handle_join', 'chat_message_join', 'message_attachment_join', 'handle', 'message', 'chat', 'attachment', 'sync_deleted_messages', 'message_processing_task', 'sync_deleted_chats', 'sync_deleted_attachments', 'kvtable', 'sqlite_stat1']


The contents of these tables should be self-explanatory.

* `attachment` keeps track of any attachments (files, images, audio clips) sent or received, paths to where they are stored, and file format.
* `handle` keeps track of all known recipients (people with whom you previously exchanged iMessages).
* `chat` keeps track of your conversation threads.
* `message` keeps track of all messages along with their text contents, date, and the ID of the recipient.
* `chat_handle_join`, `chat_message_join`, and `message_attachment_join` are all used for joining tables.

While investigating the tables, I found a field in the `message` table that I wanted to explore. 


```python
# print chat table schema
message_table = table_col_info(sql_command, 'chat', print_out=True)
print(message_table)
```

    
    Column Info:
    ID, Name, Type, NotNull, DefaultVal, PrimaryKey
    (0, 'ROWID', 'INTEGER', 0, None, 1)
    (1, 'guid', 'TEXT', 1, None, 0)
    (2, 'style', 'INTEGER', 0, None, 0)
    (3, 'state', 'INTEGER', 0, None, 0)
    (4, 'account_id', 'TEXT', 0, None, 0)
    (5, 'properties', 'BLOB', 0, None, 0)
    (6, 'chat_identifier', 'TEXT', 0, None, 0)
    (7, 'service_name', 'TEXT', 0, None, 0)
    (8, 'room_name', 'TEXT', 0, None, 0)
    (9, 'account_login', 'TEXT', 0, None, 0)
    (10, 'is_archived', 'INTEGER', 0, '0', 0)
    (11, 'last_addressed_handle', 'TEXT', 0, None, 0)
    (12, 'display_name', 'TEXT', 0, None, 0)
    (13, 'group_id', 'TEXT', 0, None, 0)
    (14, 'is_filtered', 'INTEGER', 0, None, 0)
    (15, 'successful_query', 'INTEGER', 0, None, 0)
    (16, 'engram_id', 'TEXT', 0, None, 0)
    (17, 'server_change_token', 'TEXT', 0, None, 0)
    (18, 'ck_sync_state', 'INTEGER', 0, '0', 0)
    (19, 'last_read_message_timestamp', 'INTEGER', 0, '0', 0)
    (20, 'ck_record_system_property_blob', 'BLOB', 0, None, 0)
    (21, 'original_group_id', 'TEXT', 0, 'NULL', 0)
    (22, 'sr_server_change_token', 'TEXT', 0, None, 0)
    (23, 'sr_ck_sync_state', 'INTEGER', 0, '0', 0)
    (24, 'cloudkit_record_id', 'TEXT', 0, 'NULL', 0)
    (25, 'sr_cloudkit_record_id', 'TEXT', 0, 'NULL', 0)
    None


### The field that caught my eye is `is_read`. 

How this field works:
> Apple's iMessage app provides message status updates that let you know when a message has been delivered. It also has a handy feature called Read Receipts that lets you know when someone has read the message.

As you might know, you can disable and enable Read Reciepts on the iMessage app. I wonder if you disable them, can someone still see if their message has been read by looking into the database? 


```python
# query to investigate is_read field 
is_read_query = '''
                SELECT ROWID, guid, text, is_read, is_sent, is_from_me, message_source,
                datetime(substr(date, 1, 9) + 978307200, 'unixepoch', 'localtime') as date
                FROM message T1
                WHERE T1.handle_id=3
                ORDER BY T1.date;
                '''

# execute sql query in sqlite3 
table = sql_command.execute(is_read_query)

# table.description is description of columns
column_names = get_col_names(table)

# load sql table into pandas dataframe and pass in the column names
is_read_df = pd.DataFrame(table.fetchall(), columns = column_names)

is_read_df.tail(5)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>ROWID</th>
      <th>guid</th>
      <th>text</th>
      <th>is_read</th>
      <th>is_sent</th>
      <th>is_from_me</th>
      <th>message_source</th>
      <th>date</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>29511</th>
      <td>37272</td>
      <td>BCB57A6F-28C2-4EB2-83A4-37C3718D4145</td>
      <td>How are you doing?</td>
      <td>0</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>2018-04-23 07:14:34</td>
    </tr>
    <tr>
      <th>29512</th>
      <td>37276</td>
      <td>B4AFAD0E-79CA-4605-916A-8D6A95E9BD74</td>
      <td>I hope you have a great day!</td>
      <td>0</td>
      <td>1</td>
      <td>1</td>
      <td>0</td>
      <td>2018-04-23 07:36:52</td>
    </tr>
    <tr>
      <th>29513</th>
      <td>37278</td>
      <td>6502465B-5F78-42DE-A130-DFFFFE8A8B66</td>
      <td>Hi! Thanks our day is almost over lol</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>2018-04-23 07:37:00</td>
    </tr>
    <tr>
      <th>29514</th>
      <td>37279</td>
      <td>CD977B68-198F-4480-B3D6-63BC2B2464CE</td>
      <td>Have a great Monday!</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>2018-04-23 07:37:06</td>
    </tr>
    <tr>
      <th>29515</th>
      <td>37281</td>
      <td>B5C02F7A-783C-4980-AB8A-C02C1A8825DC</td>
      <td>We’re in a Jewish restaurant rn</td>
      <td>1</td>
      <td>0</td>
      <td>0</td>
      <td>0</td>
      <td>2018-04-23 07:37:21</td>
    </tr>
  </tbody>
</table>
</div>



### My assumption is  wrong!

As you can see, I've included the Read Receipt information on my conversation. 

I sent the message **I hope you have a great day!** and even after *Birdget* responded, the **is_read** field is still 0. However, the messages that I received have a value of 1. So the is_read is actually saying if I read the message or not. 

## Extract data
-----------


```python
# query to get all messages and attachments for a specific conversation
message_query = '''
                SELECT ROWID, is_from_me, text, handle_id, cache_has_attachments,
                datetime(substr(date, 1, 9) + 978307200, 'unixepoch', 'localtime') as date
                FROM message T1
                INNER JOIN chat_message_join T2
                    ON T2.chat_id=3
                    AND T1.ROWID=T2.message_id
                ORDER BY T1.date;
                '''
# execute sql query in sqlite3 
message_table = sql_command.execute(message_query)

# get list of message column names 
message_table_columns = get_col_names(message_table)

# load sql table into pandas dataframe and pass in the column names
message_df = pd.DataFrame(message_table.fetchall(), columns = message_table_columns)

attachment_query = '''
                   SELECT T1.ROWID, T2.mime_type
                   FROM message T1
                   INNER JOIN chat_message_join T3
                       ON T1.ROWID=T3.message_id
                   INNER JOIN attachment T2
                   INNER JOIN message_attachment_join T4
                       ON T2.ROWID=T4.attachment_id
                   WHERE T4.message_id=T1.ROWID
                       AND (T3.chat_id=3);
                   '''
# execute sql query in sqlite3 
attachment_table = sql_command.execute(attachment_query)

# get list of attachment column names 
attachment_table_columns = get_col_names(attachment_table)

# load sql table into pandas dataframe and pass in the column names
attachment_df = pd.DataFrame(attachment_table.fetchall(), columns = attachment_table_columns)

# join message dataframe and attachment dataframe on ROWID 
chat_df = pd.merge(message_df, attachment_df, how = 'left', on = 'ROWID')

# close sql connection
close(sqlite_db)

# print out the last 5 rows
chat_df.tail(5)

```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>ROWID</th>
      <th>is_from_me</th>
      <th>text</th>
      <th>handle_id</th>
      <th>cache_has_attachments</th>
      <th>date</th>
      <th>mime_type</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>29701</th>
      <td>37272</td>
      <td>1</td>
      <td>How are you doing?</td>
      <td>3</td>
      <td>0</td>
      <td>2018-04-23 07:14:34</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>29702</th>
      <td>37276</td>
      <td>1</td>
      <td>I hope you have a great day!</td>
      <td>3</td>
      <td>0</td>
      <td>2018-04-23 07:36:52</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>29703</th>
      <td>37278</td>
      <td>0</td>
      <td>Hi! Thanks our day is almost over lol</td>
      <td>3</td>
      <td>0</td>
      <td>2018-04-23 07:37:00</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>29704</th>
      <td>37279</td>
      <td>0</td>
      <td>Have a great Monday!</td>
      <td>3</td>
      <td>0</td>
      <td>2018-04-23 07:37:06</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>29705</th>
      <td>37281</td>
      <td>0</td>
      <td>We’re in a Jewish restaurant rn</td>
      <td>3</td>
      <td>0</td>
      <td>2018-04-23 07:37:21</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>



## Transform data


```python
# house cleaning - drop duplicate columns from dataframe
chat_df.drop(['ROWID'], axis = 1, inplace = True)

# rename columns 
chat_df.rename(columns={'handle_id':'message_id',
                        'is_from_me':'is_sent', 
                        'text':'message',
                        'date':'message_date',
                        'cache_has_attachments':'has_attachment', 
                        'mime_type':'attachment_type'}, inplace = True)

# show the last 5 rows with new column names
chat_df.tail(5)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>is_sent</th>
      <th>message</th>
      <th>message_id</th>
      <th>has_attachment</th>
      <th>message_date</th>
      <th>attachment_type</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>29701</th>
      <td>1</td>
      <td>How are you doing?</td>
      <td>3</td>
      <td>0</td>
      <td>2018-04-23 07:14:34</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>29702</th>
      <td>1</td>
      <td>I hope you have a great day!</td>
      <td>3</td>
      <td>0</td>
      <td>2018-04-23 07:36:52</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>29703</th>
      <td>0</td>
      <td>Hi! Thanks our day is almost over lol</td>
      <td>3</td>
      <td>0</td>
      <td>2018-04-23 07:37:00</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>29704</th>
      <td>0</td>
      <td>Have a great Monday!</td>
      <td>3</td>
      <td>0</td>
      <td>2018-04-23 07:37:06</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>29705</th>
      <td>0</td>
      <td>We’re in a Jewish restaurant rn</td>
      <td>3</td>
      <td>0</td>
      <td>2018-04-23 07:37:21</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>




```python
# add is_received column
# it's the opposite of is_sent column. exmaple: is_sent = 1, is_received = 0
chat_df['is_received'] = chat_df.apply(lambda row: int(not bool(row.is_sent)), axis = 1)
```


```python
# transform message_date column into datetime.datetime object 
chat_df['message_date'] = pd.to_datetime(chat_df['message_date'])
```


```python
def add_name_col(is_sent):
    '''return name depending on is_sent column value'''
    if bool(is_sent):
        return 'Adam'
    return 'Bridget'

chat_df['name'] = chat_df.apply(lambda row: add_name_col(row.is_sent), axis = 1)

chat_df.tail(5)
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>is_sent</th>
      <th>message</th>
      <th>message_id</th>
      <th>has_attachment</th>
      <th>message_date</th>
      <th>attachment_type</th>
      <th>is_received</th>
      <th>name</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>29701</th>
      <td>1</td>
      <td>How are you doing?</td>
      <td>3</td>
      <td>0</td>
      <td>2018-04-23 07:14:34</td>
      <td>NaN</td>
      <td>0</td>
      <td>Adam</td>
    </tr>
    <tr>
      <th>29702</th>
      <td>1</td>
      <td>I hope you have a great day!</td>
      <td>3</td>
      <td>0</td>
      <td>2018-04-23 07:36:52</td>
      <td>NaN</td>
      <td>0</td>
      <td>Adam</td>
    </tr>
    <tr>
      <th>29703</th>
      <td>0</td>
      <td>Hi! Thanks our day is almost over lol</td>
      <td>3</td>
      <td>0</td>
      <td>2018-04-23 07:37:00</td>
      <td>NaN</td>
      <td>1</td>
      <td>Bridget</td>
    </tr>
    <tr>
      <th>29704</th>
      <td>0</td>
      <td>Have a great Monday!</td>
      <td>3</td>
      <td>0</td>
      <td>2018-04-23 07:37:06</td>
      <td>NaN</td>
      <td>1</td>
      <td>Bridget</td>
    </tr>
    <tr>
      <th>29705</th>
      <td>0</td>
      <td>We’re in a Jewish restaurant rn</td>
      <td>3</td>
      <td>0</td>
      <td>2018-04-23 07:37:21</td>
      <td>NaN</td>
      <td>1</td>
      <td>Bridget</td>
    </tr>
  </tbody>
</table>
</div>



## Visualize data


```python
# pie chart for the number of messages sent between us
def plot_pie(labels, values):
    fig = {
        "data":
        [
          {
              "values": values,
              "labels": labels,
              "name": "Messages Betweeen Us",
              "hoverinfo":"label+percent",
              "textinfo":"percent+value",
              "hole": 0,
              "type": "pie",
              "marker": {"line": 
                         {'color':'#000000',
                          'width':2
                         }
                        }
          }
      ],
        "layout": {
            "title":"Number of Messages",
            "annotations": [
                {
                    "font": {"size": 10},
                    "showarrow": False,
                    "text": ""
                }
            ],
      }
    }
    iplot(fig, config = {'displayModeBar': False, 'showLink': False})

    
messages_labels = ['Adam', 'Bridget']
messages_values = [chat_df['is_sent'].value_counts()[1], 
                   chat_df['is_received'].value_counts()[1]]
plot_pie(messages_labels, messages_values)
```


<div id="f7a9f487-0ef0-49dd-9f6b-5798af61eb47" style="height: 525px; width: 100%;" class="plotly-graph-div"></div><script type="text/javascript">require(["plotly"], function(Plotly) { window.PLOTLYENV=window.PLOTLYENV || {};window.PLOTLYENV.BASE_URL="https://plot.ly";Plotly.newPlot("f7a9f487-0ef0-49dd-9f6b-5798af61eb47", [{"values": [15434, 14272], "labels": ["Adam", "Bridget"], "name": "Messages Betweeen Us", "hoverinfo": "label+percent", "textinfo": "percent+value", "hole": 0, "type": "pie", "marker": {"line": {"color": "#000000", "width": 2}}}], {"title": "Number of Messages", "annotations": [{"font": {"size": 10}, "showarrow": false, "text": ""}]}, {"showLink": false, "linkText": "Export to plot.ly", "displayModeBar": false})});</script>



```python
exclude = set(string.punctuation)
def remove_punctuation(x):
    '''
    function to remove punctuation from a string
    x: any string
    '''
    try:
        x = x.translate(str.maketrans("", "", string.punctuation))
    except:
        pass
    return x
    
# remove attachment/picture messages, and null message values
word_count_df = chat_df[chat_df['has_attachment'] != 1] 
word_count_df = word_count_df[word_count_df['message'].isnull() != True]
word_count_df = word_count_df['message'].to_frame()

# remove punctuation and backwards apostrophe
word_count_df['message'] = word_count_df['message'].apply(lambda x: remove_punctuation(x)) 
word_count_df['message'] = word_count_df['message'].str.replace(r"’", '') 

# lower case all messages and split words on spaces 
word_count_df['message'] = word_count_df['message'].str.lower().str.split(' ')

# flatten nested list to create list of every word 
message_list = [word for sublist in word_count_df.message.tolist() for word in sublist]

# count most used words
words_count = {}
for word in message_list:
    if word in words_count:
        words_count[word] += 1
    else:
        words_count[word] = 1
        
# create list of tuples and sort words by highiest count
words_count = sorted(words_count.items(), key = lambda x: x[1], reverse = True)

# get counts of every word into a dataframe
word_count_df = pd.DataFrame(words_count, columns = ['word', 'count'])

# remove blank values from word column (i.e. space or empty messages)
word_count_df = word_count_df[word_count_df['word'] != '']

# bar chart for top 10 words sent between us
def plot_bar(labels, values, xaxis, yaxis):
    trace = go.Bar(
        x = labels,
        y = values,
        marker = dict(
            line = dict(
                color = '#00000)',
                width = 2,
            )
        ),
    )
    data = [trace]
    layout = go.Layout(
        title = 'Top 10 Words',
        xaxis = dict(
            title=xaxis,
            autorange = True,
            showgrid = False,
            zeroline = False,
            showline = True,
            autotick = False,
            ticks = labels,
            showticklabels = True
        ),
        yaxis = dict(
            title=yaxis,
            autorange = True,
            showgrid = True,
            zeroline = False,
            showline = True,
            autotick = True,
            ticks = '',
            showticklabels = True
        )
    )
    fig = go.Figure(data = data, layout = layout)
    iplot(fig, config = {'displayModeBar': False, 'showLink': False})
    
# only chart top 10
word_count_df = word_count_df.head(10)
labels = word_count_df['word'].values.tolist()
values = word_count_df['count'].values.tolist()
plot_bar(labels, values, "Word", "Count")
```


<div id="0528f8a3-ff50-4d53-9829-86872f903818" style="height: 525px; width: 100%;" class="plotly-graph-div"></div><script type="text/javascript">require(["plotly"], function(Plotly) { window.PLOTLYENV=window.PLOTLYENV || {};window.PLOTLYENV.BASE_URL="https://plot.ly";Plotly.newPlot("0528f8a3-ff50-4d53-9829-86872f903818", [{"type": "bar", "x": ["you", "i", "to", "bb", "the", "a", "and", "im", "are", "it"], "y": [5446, 3767, 3261, 2274, 2208, 1685, 1650, 1421, 1364, 1348], "marker": {"line": {"color": "#00000)", "width": 2}}}], {"title": "Top 10 Words", "xaxis": {"title": "Word", "autorange": true, "showgrid": false, "zeroline": false, "showline": true, "autotick": false, "ticks": ["you", "i", "to", "bb", "the", "a", "and", "im", "are", "it"], "showticklabels": true}, "yaxis": {"title": "Count", "autorange": true, "showgrid": true, "zeroline": false, "showline": true, "autotick": true, "ticks": "", "showticklabels": true}}, {"showLink": false, "linkText": "Export to plot.ly", "displayModeBar": false})});</script>



```python
months_map = {1:'Jan', 2:'Feb', 3:'Mar', 4:'Apr', 5:'May', 6:'Jun',
              7:'Jul', 8:'Aug', 9:'Sep', 10:'Oct', 11:'Nov', 12:'Dec'}
def get_month(month):
    '''helper function for getting a three letter month for pandas month value'''
    return months_map[month]

# create dataframes for each of us 
adam_df = chat_df[chat_df.is_sent == 1]
bridget_df = chat_df[chat_df.is_received == 1]

# group messages by year and month and counts
adam_df = adam_df['message_date'].groupby([adam_df.message_date.dt.year, adam_df.message_date.dt.month]).agg('count')
bridget_df = bridget_df['message_date'].groupby([bridget_df.message_date.dt.year, bridget_df.message_date.dt.month]).agg('count')

# rename grouped columns and make series in to a dataframe
adam_df = pd.DataFrame({'month':adam_df.index, 'count_a':adam_df.values}) 
bridget_df = pd.DataFrame({'month':bridget_df.index, 'count_b':bridget_df.values}) 

# get three letter month
adam_df['month'] = [get_month(x[1]) for x in adam_df.month.values] 
bridget_df['month'] = [get_month(x[1]) for x in bridget_df.month.values]

# merge with adam dataframe to create a single dataframe
month_count_df = pd.merge(adam_df, bridget_df, how = 'inner', on = 'month')
    
def plot_group_bar(labels, value_a, value_b, title, xaxis, yaxis):
    trace_1 = go.Bar(
        x = labels,
        y = value_a,
        name='Adam',
        marker = dict(
            line = dict(
                color = '#00000)',
                width = 2,
            )
        ),
    )
    
    trace_2 = go.Bar(
        x = labels,
        y = value_b,
        name='Bridget',
        marker = dict(
            line = dict(
                color = '#00000)',
                width = 2,
            )
        ),
    )
    data = [trace_1, trace_2]
    layout = go.Layout(
        title = title,
        barmode = 'group',
        xaxis = dict(
            title=xaxis,
            autorange = True,
            showgrid = False,
            zeroline = False,
            showline = True,
            autotick = False,
            ticks = labels,
            showticklabels = True
        ),
        yaxis = dict(
            title=yaxis,
            autorange = True,
            showgrid = True,
            zeroline = False,
            showline = True,
            autotick = True,
            ticks = '',
            showticklabels = True
        )
    )
    fig = go.Figure(data = data, layout = layout)
    iplot(fig, config = {'displayModeBar': False, 'showLink': False})
    
labels = month_count_df['month'].values.tolist()
a_count = month_count_df['count_a'].values.tolist()
b_count = month_count_df['count_b'].values.tolist()
title = 'Messages by Month'
plot_group_bar(labels, a_count, b_count, title, 'Month', "Messages")
```


<div id="0758af1d-3eb2-4124-a5f9-4d76154cafce" style="height: 525px; width: 100%;" class="plotly-graph-div"></div><script type="text/javascript">require(["plotly"], function(Plotly) { window.PLOTLYENV=window.PLOTLYENV || {};window.PLOTLYENV.BASE_URL="https://plot.ly";Plotly.newPlot("0758af1d-3eb2-4124-a5f9-4d76154cafce", [{"type": "bar", "x": ["Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec", "Jan", "Feb", "Mar", "Apr"], "y": [825, 1077, 1902, 1192, 1184, 1436, 1952, 1456, 1840, 1466, 1104], "name": "Adam", "marker": {"line": {"color": "#00000)", "width": 2}}}, {"type": "bar", "x": ["Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec", "Jan", "Feb", "Mar", "Apr"], "y": [693, 1003, 1892, 1193, 1188, 1350, 1741, 1282, 1623, 1301, 1006], "name": "Bridget", "marker": {"line": {"color": "#00000)", "width": 2}}}], {"title": "Messages by Month", "barmode": "group", "xaxis": {"title": "Month", "autorange": true, "showgrid": false, "zeroline": false, "showline": true, "autotick": false, "ticks": ["Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec", "Jan", "Feb", "Mar", "Apr"], "showticklabels": true}, "yaxis": {"title": "Messages", "autorange": true, "showgrid": true, "zeroline": false, "showline": true, "autotick": true, "ticks": "", "showticklabels": true}}, {"showLink": false, "linkText": "Export to plot.ly", "displayModeBar": false})});</script>



```python
week_map = {0:'Mon', 1:'Tue', 2:'Wed', 3:'Thu', 
            4:'Fri', 5:'Sat', 6:'Sun'}
def get_weekday(day):
    '''helper function for getting a three letter weekday for pandas weekday value'''
    return week_map[day]

# create dataframes for each of us 
adam_df = chat_df[chat_df.is_sent == 1]
bridget_df = chat_df[chat_df.is_received == 1]

# group messages by weekday and counts
adam_df = adam_df['message_date'].groupby([adam_df.message_date.dt.weekday]).agg('count')
bridget_df = bridget_df['message_date'].groupby([bridget_df.message_date.dt.weekday]).agg('count') 

# rename grouped columns and make series in to a dataframe
adam_df = pd.DataFrame({'weekday':adam_df.index, 'count_a':adam_df.values}) 
bridget_df = pd.DataFrame({'weekday':bridget_df.index, 'count_b':bridget_df.values})

# get three letter weekday 
adam_df['weekday'] = adam_df.apply(lambda row: get_weekday(row.weekday), axis = 1) 
bridget_df['weekday'] = bridget_df.apply(lambda row: get_weekday(row.weekday), axis = 1)

# merge with adam dataframe to create a single dataframe
week_count_df = pd.merge(adam_df, bridget_df, how = 'inner', on = 'weekday')


# plot group bar chart for messages by weekday 
labels = week_count_df['weekday'].values.tolist()
a_count = week_count_df['count_a'].values.tolist()
b_count = week_count_df['count_b'].values.tolist()
title = 'Messages by Weekday'
plot_group_bar(labels, a_count, b_count, title, "Weekday", "Messages")
```


<div id="f2951a13-1986-4e37-bb8a-686b5de6e413" style="height: 525px; width: 100%;" class="plotly-graph-div"></div><script type="text/javascript">require(["plotly"], function(Plotly) { window.PLOTLYENV=window.PLOTLYENV || {};window.PLOTLYENV.BASE_URL="https://plot.ly";Plotly.newPlot("f2951a13-1986-4e37-bb8a-686b5de6e413", [{"type": "bar", "x": ["Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"], "y": [2334, 2395, 2472, 2177, 2258, 1931, 1867], "name": "Adam", "marker": {"line": {"color": "#00000)", "width": 2}}}, {"type": "bar", "x": ["Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"], "y": [2085, 2164, 2208, 2129, 2162, 1810, 1714], "name": "Bridget", "marker": {"line": {"color": "#00000)", "width": 2}}}], {"title": "Messages by Weekday", "barmode": "group", "xaxis": {"title": "Weekday", "autorange": true, "showgrid": false, "zeroline": false, "showline": true, "autotick": false, "ticks": ["Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"], "showticklabels": true}, "yaxis": {"title": "Messages", "autorange": true, "showgrid": true, "zeroline": false, "showline": true, "autotick": true, "ticks": "", "showticklabels": true}}, {"showLink": false, "linkText": "Export to plot.ly", "displayModeBar": false})});</script>



```python
# message by hour
# create dataframes for each of us 
adam_df = chat_df[chat_df.is_sent == 1]
bridget_df = chat_df[chat_df.is_received == 1]

# group messages by weekday and counts
adam_df = adam_df['message_date'].groupby([adam_df.message_date.dt.hour]).agg('count')
bridget_df = bridget_df['message_date'].groupby([bridget_df.message_date.dt.hour]).agg('count') 

# rename grouped columns and make series in to a dataframe
adam_df = pd.DataFrame({'hour':adam_df.index, 'count_a':adam_df.values}) 
bridget_df = pd.DataFrame({'hour':bridget_df.index, 'count_b':bridget_df.values})

# merge with adam dataframe to create a single dataframe
hour_count_df = pd.merge(adam_df, bridget_df, how = 'inner', on = 'hour')

# plot group bar chart for messages by hour 
labels = hour_count_df['hour'].values.tolist()
a_count = hour_count_df['count_a'].values.tolist()
b_count = hour_count_df['count_b'].values.tolist()
title = 'Messages by Hour of Day'
plot_group_bar(labels, a_count, b_count, title, "Hour", "Messages")
```


<div id="50d5d003-45df-4032-90d5-3767b10e1297" style="height: 525px; width: 100%;" class="plotly-graph-div"></div><script type="text/javascript">require(["plotly"], function(Plotly) { window.PLOTLYENV=window.PLOTLYENV || {};window.PLOTLYENV.BASE_URL="https://plot.ly";Plotly.newPlot("50d5d003-45df-4032-90d5-3767b10e1297", [{"type": "bar", "x": [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23], "y": [235, 178, 25, 1, 4, 10, 174, 454, 615, 751, 683, 622, 664, 825, 857, 1119, 1465, 1526, 1160, 1012, 873, 1018, 715, 448], "name": "Adam", "marker": {"line": {"color": "#00000)", "width": 2}}}, {"type": "bar", "x": [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23], "y": [164, 134, 30, 4, 13, 9, 175, 415, 640, 672, 591, 644, 646, 697, 849, 1084, 1358, 1507, 1137, 1005, 771, 763, 637, 327], "name": "Bridget", "marker": {"line": {"color": "#00000)", "width": 2}}}], {"title": "Messages by Hour of Day", "barmode": "group", "xaxis": {"title": "Hour", "autorange": true, "showgrid": false, "zeroline": false, "showline": true, "autotick": false, "ticks": [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23], "showticklabels": true}, "yaxis": {"title": "Messages", "autorange": true, "showgrid": true, "zeroline": false, "showline": true, "autotick": true, "ticks": "", "showticklabels": true}}, {"showLink": false, "linkText": "Export to plot.ly", "displayModeBar": false})});</script>



```python
# Get message over time
adam_df = chat_df[chat_df.is_sent == 1]
bridget_df = chat_df[chat_df.is_received == 1]

# group messages by month, day, year  and counts
adam_df = adam_df['message_date'].groupby([adam_df.message_date.dt.date]).agg('count')
bridget_df = bridget_df['message_date'].groupby([bridget_df.message_date.dt.date]).agg('count')

# rename grouped columns and make series in to a dataframe
adam_df = pd.DataFrame({'date':adam_df.index, 'count_a':adam_df.values}) 
bridget_df = pd.DataFrame({'date':bridget_df.index, 'count_b':bridget_df.values})

# merge dataframes to create a single dataframe
date_count_df = pd.merge(adam_df, bridget_df, how = 'inner', on = 'date')

def plot_scatter(labels, value_a, value_b, title, xaxis, yaxis):
    trace_1 = go.Scatter(
        x = labels,
        y = value_a,
        name='Adam',
        fill = 'tonexty',
        mode= 'lines'
    )
    
    trace_2 = go.Scatter(
        x = labels,
        y = value_b,
        name='Bridget',
        fill = 'none',
        mode= 'lines'
    )
    data = [trace_1, trace_2]
    layout = go.Layout(
        title = title,
        xaxis = dict(
            title = xaxis,
            autorange = True,
            showgrid = True,
            zeroline = True,
            showline = False,
            autotick = True,
            ticks = '',
            showticklabels = True
        ),
        yaxis = dict(
            title = yaxis,
            autorange = True,
            showgrid = True,
            zeroline = True,
            showline = True,
            autotick = True,
            ticks = '',
            showticklabels = True
        )
    )
    fig = go.Figure(data = data, layout = layout)
    iplot(fig, config = {'displayModeBar': False, 'showLink': False})

# plot scatter chart for messages over time 
labels = date_count_df['date'].values.tolist()
a_count = date_count_df['count_a'].values.tolist()
b_count = date_count_df['count_b'].values.tolist()
title = 'Messages by Over Time'
plot_scatter(labels, a_count, b_count, title, "Date", "Messages")
```


<div id="099567bf-8b3d-4ba1-89d0-15f39b11fcbe" style="height: 525px; width: 100%;" class="plotly-graph-div"></div><script type="text/javascript">require(["plotly"], function(Plotly) { window.PLOTLYENV=window.PLOTLYENV || {};window.PLOTLYENV.BASE_URL="https://plot.ly";Plotly.newPlot("099567bf-8b3d-4ba1-89d0-15f39b11fcbe", [{"type": "scatter", "x": ["2017-06-16", "2017-06-17", "2017-06-18", "2017-06-19", "2017-06-20", "2017-06-21", "2017-06-22", "2017-06-23", "2017-06-24", "2017-06-25", "2017-06-26", "2017-06-27", "2017-06-28", "2017-06-29", "2017-06-30", "2017-07-01", "2017-07-16", "2017-07-17", "2017-07-18", "2017-07-19", "2017-07-20", "2017-07-21", "2017-07-22", "2017-07-24", "2017-07-25", "2017-07-26", "2017-07-27", "2017-07-28", "2017-07-29", "2017-07-30", "2017-07-31", "2017-08-01", "2017-08-02", "2017-08-03", "2017-08-04", "2017-08-05", "2017-08-06", "2017-08-07", "2017-08-08", "2017-08-09", "2017-08-10", "2017-08-11", "2017-08-12", "2017-08-13", "2017-08-14", "2017-08-15", "2017-08-16", "2017-08-17", "2017-08-18", "2017-08-19", "2017-08-21", "2017-08-22", "2017-08-23", "2017-08-24", "2017-08-25", "2017-08-26", "2017-08-27", "2017-08-28", "2017-08-29", "2017-08-30", "2017-08-31", "2017-09-01", "2017-09-02", "2017-09-03", "2017-09-04", "2017-09-05", "2017-09-06", "2017-09-07", "2017-09-08", "2017-09-09", "2017-09-10", "2017-09-11", "2017-09-12", "2017-09-13", "2017-09-14", "2017-09-15", "2017-09-16", "2017-09-17", "2017-09-18", "2017-09-19", "2017-09-20", "2017-09-21", "2017-09-22", "2017-09-23", "2017-09-24", "2017-09-25", "2017-09-26", "2017-09-27", "2017-09-28", "2017-09-29", "2017-09-30", "2017-10-01", "2017-10-02", "2017-10-03", "2017-10-04", "2017-10-05", "2017-10-06", "2017-10-07", "2017-10-08", "2017-10-10", "2017-10-11", "2017-10-12", "2017-10-13", "2017-10-14", "2017-10-15", "2017-10-16", "2017-10-17", "2017-10-18", "2017-10-19", "2017-10-20", "2017-10-21", "2017-10-22", "2017-10-23", "2017-10-24", "2017-10-25", "2017-10-26", "2017-10-27", "2017-10-28", "2017-10-29", "2017-10-30", "2017-10-31", "2017-11-01", "2017-11-02", "2017-11-03", "2017-11-04", "2017-11-05", "2017-11-06", "2017-11-07", "2017-11-08", "2017-11-09", "2017-11-10", "2017-11-11", "2017-11-12", "2017-11-13", "2017-11-14", "2017-11-15", "2017-11-16", "2017-11-17", "2017-11-18", "2017-11-19", "2017-11-20", "2017-11-21", "2017-11-22", "2017-11-23", "2017-11-24", "2017-11-25", "2017-11-26", "2017-11-27", "2017-11-28", "2017-11-29", "2017-11-30", "2017-12-01", "2017-12-02", "2017-12-03", "2017-12-04", "2017-12-05", "2017-12-06", "2017-12-07", "2017-12-08", "2017-12-09", "2017-12-10", "2017-12-11", "2017-12-12", "2017-12-13", "2017-12-14", "2017-12-15", "2017-12-16", "2017-12-17", "2017-12-18", "2017-12-19", "2017-12-20", "2017-12-21", "2017-12-22", "2017-12-23", "2017-12-24", "2017-12-25", "2017-12-26", "2017-12-27", "2017-12-28", "2017-12-29", "2017-12-30", "2017-12-31", "2018-01-01", "2018-01-02", "2018-01-03", "2018-01-04", "2018-01-05", "2018-01-06", "2018-01-07", "2018-01-08", "2018-01-09", "2018-01-10", "2018-01-11", "2018-01-12", "2018-01-13", "2018-01-14", "2018-01-15", "2018-01-16", "2018-01-17", "2018-01-18", "2018-01-19", "2018-01-20", "2018-01-21", "2018-01-22", "2018-01-23", "2018-01-24", "2018-01-25", "2018-01-26", "2018-01-27", "2018-01-28", "2018-01-29", "2018-01-30", "2018-01-31", "2018-02-01", "2018-02-02", "2018-02-03", "2018-02-04", "2018-02-05", "2018-02-06", "2018-02-07", "2018-02-08", "2018-02-09", "2018-02-10", "2018-02-11", "2018-02-12", "2018-02-13", "2018-02-14", "2018-02-15", "2018-02-16", "2018-02-17", "2018-02-18", "2018-02-19", "2018-02-20", "2018-02-21", "2018-02-22", "2018-02-23", "2018-02-24", "2018-02-25", "2018-02-26", "2018-02-27", "2018-02-28", "2018-03-01", "2018-03-02", "2018-03-03", "2018-03-04", "2018-03-05", "2018-03-06", "2018-03-07", "2018-03-08", "2018-03-09", "2018-03-10", "2018-03-11", "2018-03-12", "2018-03-13", "2018-03-14", "2018-03-15", "2018-03-16", "2018-03-17", "2018-03-18", "2018-03-19", "2018-03-20", "2018-03-21", "2018-03-22", "2018-03-23", "2018-03-24", "2018-03-25", "2018-03-26", "2018-03-27", "2018-03-28", "2018-03-29", "2018-03-30", "2018-03-31", "2018-04-01", "2018-04-02", "2018-04-03", "2018-04-04", "2018-04-05", "2018-04-06", "2018-04-07", "2018-04-08", "2018-04-09", "2018-04-10", "2018-04-11", "2018-04-12", "2018-04-15", "2018-04-16", "2018-04-17", "2018-04-18", "2018-04-19", "2018-04-20", "2018-04-21", "2018-04-22", "2018-04-23"], "y": [39, 21, 20, 40, 32, 45, 74, 3, 2, 1, 231, 43, 98, 41, 135, 74, 6, 38, 40, 50, 75, 98, 93, 68, 50, 81, 101, 88, 118, 48, 49, 96, 52, 93, 26, 56, 46, 60, 35, 65, 51, 51, 45, 25, 113, 99, 176, 59, 69, 84, 86, 77, 58, 29, 53, 98, 25, 39, 60, 31, 45, 91, 34, 11, 31, 40, 26, 55, 38, 110, 22, 27, 44, 17, 40, 46, 14, 21, 35, 24, 96, 24, 34, 1, 22, 33, 35, 47, 47, 81, 46, 50, 39, 40, 50, 78, 55, 134, 19, 41, 74, 25, 28, 6, 55, 69, 63, 49, 38, 20, 39, 7, 26, 21, 19, 25, 18, 10, 15, 29, 42, 35, 28, 120, 47, 45, 43, 46, 25, 48, 20, 19, 33, 45, 72, 20, 33, 76, 37, 58, 31, 90, 81, 7, 42, 76, 97, 46, 9, 56, 51, 65, 16, 13, 28, 56, 33, 17, 11, 3, 42, 15, 53, 33, 57, 44, 12, 62, 37, 83, 99, 52, 118, 202, 410, 92, 98, 41, 64, 49, 10, 37, 3, 29, 55, 59, 46, 51, 40, 45, 35, 12, 41, 59, 32, 33, 90, 87, 55, 31, 33, 46, 5, 26, 30, 42, 36, 28, 32, 28, 39, 112, 196, 62, 45, 42, 17, 69, 74, 83, 103, 31, 22, 45, 78, 73, 48, 59, 59, 27, 20, 33, 66, 78, 130, 28, 12, 85, 276, 102, 73, 74, 77, 33, 57, 28, 51, 50, 30, 98, 32, 45, 74, 99, 74, 63, 90, 74, 15, 30, 42, 15, 51, 12, 7, 43, 13, 28, 68, 28, 17, 48, 9, 106, 62, 62, 66, 57, 31, 91, 10, 44, 70, 58, 96, 53, 72, 34, 29, 60, 33, 48, 11], "name": "Adam", "fill": "tonexty", "mode": "lines"}, {"type": "scatter", "x": ["2017-06-16", "2017-06-17", "2017-06-18", "2017-06-19", "2017-06-20", "2017-06-21", "2017-06-22", "2017-06-23", "2017-06-24", "2017-06-25", "2017-06-26", "2017-06-27", "2017-06-28", "2017-06-29", "2017-06-30", "2017-07-01", "2017-07-16", "2017-07-17", "2017-07-18", "2017-07-19", "2017-07-20", "2017-07-21", "2017-07-22", "2017-07-24", "2017-07-25", "2017-07-26", "2017-07-27", "2017-07-28", "2017-07-29", "2017-07-30", "2017-07-31", "2017-08-01", "2017-08-02", "2017-08-03", "2017-08-04", "2017-08-05", "2017-08-06", "2017-08-07", "2017-08-08", "2017-08-09", "2017-08-10", "2017-08-11", "2017-08-12", "2017-08-13", "2017-08-14", "2017-08-15", "2017-08-16", "2017-08-17", "2017-08-18", "2017-08-19", "2017-08-21", "2017-08-22", "2017-08-23", "2017-08-24", "2017-08-25", "2017-08-26", "2017-08-27", "2017-08-28", "2017-08-29", "2017-08-30", "2017-08-31", "2017-09-01", "2017-09-02", "2017-09-03", "2017-09-04", "2017-09-05", "2017-09-06", "2017-09-07", "2017-09-08", "2017-09-09", "2017-09-10", "2017-09-11", "2017-09-12", "2017-09-13", "2017-09-14", "2017-09-15", "2017-09-16", "2017-09-17", "2017-09-18", "2017-09-19", "2017-09-20", "2017-09-21", "2017-09-22", "2017-09-23", "2017-09-24", "2017-09-25", "2017-09-26", "2017-09-27", "2017-09-28", "2017-09-29", "2017-09-30", "2017-10-01", "2017-10-02", "2017-10-03", "2017-10-04", "2017-10-05", "2017-10-06", "2017-10-07", "2017-10-08", "2017-10-10", "2017-10-11", "2017-10-12", "2017-10-13", "2017-10-14", "2017-10-15", "2017-10-16", "2017-10-17", "2017-10-18", "2017-10-19", "2017-10-20", "2017-10-21", "2017-10-22", "2017-10-23", "2017-10-24", "2017-10-25", "2017-10-26", "2017-10-27", "2017-10-28", "2017-10-29", "2017-10-30", "2017-10-31", "2017-11-01", "2017-11-02", "2017-11-03", "2017-11-04", "2017-11-05", "2017-11-06", "2017-11-07", "2017-11-08", "2017-11-09", "2017-11-10", "2017-11-11", "2017-11-12", "2017-11-13", "2017-11-14", "2017-11-15", "2017-11-16", "2017-11-17", "2017-11-18", "2017-11-19", "2017-11-20", "2017-11-21", "2017-11-22", "2017-11-23", "2017-11-24", "2017-11-25", "2017-11-26", "2017-11-27", "2017-11-28", "2017-11-29", "2017-11-30", "2017-12-01", "2017-12-02", "2017-12-03", "2017-12-04", "2017-12-05", "2017-12-06", "2017-12-07", "2017-12-08", "2017-12-09", "2017-12-10", "2017-12-11", "2017-12-12", "2017-12-13", "2017-12-14", "2017-12-15", "2017-12-16", "2017-12-17", "2017-12-18", "2017-12-19", "2017-12-20", "2017-12-21", "2017-12-22", "2017-12-23", "2017-12-24", "2017-12-25", "2017-12-26", "2017-12-27", "2017-12-28", "2017-12-29", "2017-12-30", "2017-12-31", "2018-01-01", "2018-01-02", "2018-01-03", "2018-01-04", "2018-01-05", "2018-01-06", "2018-01-07", "2018-01-08", "2018-01-09", "2018-01-10", "2018-01-11", "2018-01-12", "2018-01-13", "2018-01-14", "2018-01-15", "2018-01-16", "2018-01-17", "2018-01-18", "2018-01-19", "2018-01-20", "2018-01-21", "2018-01-22", "2018-01-23", "2018-01-24", "2018-01-25", "2018-01-26", "2018-01-27", "2018-01-28", "2018-01-29", "2018-01-30", "2018-01-31", "2018-02-01", "2018-02-02", "2018-02-03", "2018-02-04", "2018-02-05", "2018-02-06", "2018-02-07", "2018-02-08", "2018-02-09", "2018-02-10", "2018-02-11", "2018-02-12", "2018-02-13", "2018-02-14", "2018-02-15", "2018-02-16", "2018-02-17", "2018-02-18", "2018-02-19", "2018-02-20", "2018-02-21", "2018-02-22", "2018-02-23", "2018-02-24", "2018-02-25", "2018-02-26", "2018-02-27", "2018-02-28", "2018-03-01", "2018-03-02", "2018-03-03", "2018-03-04", "2018-03-05", "2018-03-06", "2018-03-07", "2018-03-08", "2018-03-09", "2018-03-10", "2018-03-11", "2018-03-12", "2018-03-13", "2018-03-14", "2018-03-15", "2018-03-16", "2018-03-17", "2018-03-18", "2018-03-19", "2018-03-20", "2018-03-21", "2018-03-22", "2018-03-23", "2018-03-24", "2018-03-25", "2018-03-26", "2018-03-27", "2018-03-28", "2018-03-29", "2018-03-30", "2018-03-31", "2018-04-01", "2018-04-02", "2018-04-03", "2018-04-04", "2018-04-05", "2018-04-06", "2018-04-07", "2018-04-08", "2018-04-09", "2018-04-10", "2018-04-11", "2018-04-12", "2018-04-15", "2018-04-16", "2018-04-17", "2018-04-18", "2018-04-19", "2018-04-20", "2018-04-21", "2018-04-22", "2018-04-23"], "y": [38, 22, 21, 47, 22, 49, 73, 5, 3, 15, 74, 33, 86, 57, 148, 83, 4, 33, 54, 52, 72, 93, 76, 85, 37, 60, 80, 62, 106, 38, 68, 71, 53, 114, 33, 51, 87, 60, 37, 51, 46, 80, 45, 29, 92, 95, 142, 65, 64, 72, 75, 72, 63, 33, 67, 77, 33, 42, 53, 35, 54, 79, 36, 10, 26, 42, 26, 82, 47, 116, 24, 19, 41, 24, 39, 46, 13, 19, 42, 22, 84, 46, 32, 4, 18, 33, 31, 45, 45, 73, 29, 39, 41, 47, 38, 69, 35, 101, 24, 52, 65, 22, 33, 13, 65, 61, 57, 46, 42, 44, 36, 7, 29, 28, 30, 27, 28, 11, 9, 42, 29, 45, 26, 105, 44, 46, 58, 47, 26, 41, 24, 17, 22, 45, 65, 18, 25, 64, 28, 46, 29, 83, 56, 6, 50, 81, 85, 40, 9, 60, 59, 62, 15, 12, 32, 66, 47, 18, 19, 1, 37, 19, 47, 30, 44, 50, 21, 38, 42, 63, 56, 44, 79, 166, 340, 95, 99, 55, 47, 58, 10, 29, 8, 25, 43, 64, 44, 50, 28, 45, 35, 11, 41, 45, 31, 36, 76, 72, 60, 24, 30, 43, 7, 28, 25, 47, 31, 31, 36, 37, 37, 98, 94, 59, 41, 49, 22, 58, 68, 67, 80, 32, 29, 36, 73, 64, 57, 50, 43, 32, 18, 27, 48, 94, 100, 32, 37, 68, 182, 94, 63, 68, 67, 26, 46, 32, 48, 37, 43, 81, 15, 48, 40, 73, 65, 50, 67, 56, 4, 40, 40, 18, 70, 15, 14, 37, 21, 25, 55, 35, 19, 46, 11, 88, 42, 56, 54, 41, 30, 85, 12, 53, 68, 54, 89, 58, 52, 31, 30, 56, 39, 44, 13], "name": "Bridget", "fill": "none", "mode": "lines"}], {"title": "Messages by Over Time", "xaxis": {"title": "Date", "autorange": true, "showgrid": true, "zeroline": true, "showline": false, "autotick": true, "ticks": "", "showticklabels": true}, "yaxis": {"title": "Messages", "autorange": true, "showgrid": true, "zeroline": true, "showline": true, "autotick": true, "ticks": "", "showticklabels": true}}, {"showLink": false, "linkText": "Export to plot.ly", "displayModeBar": false})});</script>



```python
# HEAT MAP !
def plot_heat(labels, values, title, xaxis, yaxis):
    data = [
        go.Heatmap(
            z = values,
            x = labels,
            y = ['Adam','Bridget'],
            colorscale =' Viridis',
            ygap = 0.5,
        )
    ]
    
    layout = go.Layout(
        title = title,
        xaxis = dict(
            title = xaxis,
            autorange = True,
            showgrid = True,
            zeroline = True,
            showline = False,
            autotick = True,
            nticks = 18,
            showticklabels = True
        ),
        yaxis = dict(
            title = yaxis,
            autorange = True,
            showgrid = True,
            zeroline = True,
            showline = True,
            autotick = True,
            ticks = '',
            showticklabels = True
        )
    )
    fig = go.Figure(data=data, layout=layout)
    iplot(fig, config = {'displayModeBar': False, 'showLink': False})
    
# plot scatter chart for messages over time 
labels = date_count_df['date'].values.tolist()
a_count = date_count_df['count_a'].values.tolist()
b_count = date_count_df['count_b'].values.tolist()
title = 'Messages by Over Time'
plot_heat(labels, [a_count, b_count], title, "Date", "Person")
```


<div id="812c45ec-05a2-49b4-9bd0-705437cd7d96" style="height: 525px; width: 100%;" class="plotly-graph-div"></div><script type="text/javascript">require(["plotly"], function(Plotly) { window.PLOTLYENV=window.PLOTLYENV || {};window.PLOTLYENV.BASE_URL="https://plot.ly";Plotly.newPlot("812c45ec-05a2-49b4-9bd0-705437cd7d96", [{"type": "heatmap", "z": [[39, 21, 20, 40, 32, 45, 74, 3, 2, 1, 231, 43, 98, 41, 135, 74, 6, 38, 40, 50, 75, 98, 93, 68, 50, 81, 101, 88, 118, 48, 49, 96, 52, 93, 26, 56, 46, 60, 35, 65, 51, 51, 45, 25, 113, 99, 176, 59, 69, 84, 86, 77, 58, 29, 53, 98, 25, 39, 60, 31, 45, 91, 34, 11, 31, 40, 26, 55, 38, 110, 22, 27, 44, 17, 40, 46, 14, 21, 35, 24, 96, 24, 34, 1, 22, 33, 35, 47, 47, 81, 46, 50, 39, 40, 50, 78, 55, 134, 19, 41, 74, 25, 28, 6, 55, 69, 63, 49, 38, 20, 39, 7, 26, 21, 19, 25, 18, 10, 15, 29, 42, 35, 28, 120, 47, 45, 43, 46, 25, 48, 20, 19, 33, 45, 72, 20, 33, 76, 37, 58, 31, 90, 81, 7, 42, 76, 97, 46, 9, 56, 51, 65, 16, 13, 28, 56, 33, 17, 11, 3, 42, 15, 53, 33, 57, 44, 12, 62, 37, 83, 99, 52, 118, 202, 410, 92, 98, 41, 64, 49, 10, 37, 3, 29, 55, 59, 46, 51, 40, 45, 35, 12, 41, 59, 32, 33, 90, 87, 55, 31, 33, 46, 5, 26, 30, 42, 36, 28, 32, 28, 39, 112, 196, 62, 45, 42, 17, 69, 74, 83, 103, 31, 22, 45, 78, 73, 48, 59, 59, 27, 20, 33, 66, 78, 130, 28, 12, 85, 276, 102, 73, 74, 77, 33, 57, 28, 51, 50, 30, 98, 32, 45, 74, 99, 74, 63, 90, 74, 15, 30, 42, 15, 51, 12, 7, 43, 13, 28, 68, 28, 17, 48, 9, 106, 62, 62, 66, 57, 31, 91, 10, 44, 70, 58, 96, 53, 72, 34, 29, 60, 33, 48, 11], [38, 22, 21, 47, 22, 49, 73, 5, 3, 15, 74, 33, 86, 57, 148, 83, 4, 33, 54, 52, 72, 93, 76, 85, 37, 60, 80, 62, 106, 38, 68, 71, 53, 114, 33, 51, 87, 60, 37, 51, 46, 80, 45, 29, 92, 95, 142, 65, 64, 72, 75, 72, 63, 33, 67, 77, 33, 42, 53, 35, 54, 79, 36, 10, 26, 42, 26, 82, 47, 116, 24, 19, 41, 24, 39, 46, 13, 19, 42, 22, 84, 46, 32, 4, 18, 33, 31, 45, 45, 73, 29, 39, 41, 47, 38, 69, 35, 101, 24, 52, 65, 22, 33, 13, 65, 61, 57, 46, 42, 44, 36, 7, 29, 28, 30, 27, 28, 11, 9, 42, 29, 45, 26, 105, 44, 46, 58, 47, 26, 41, 24, 17, 22, 45, 65, 18, 25, 64, 28, 46, 29, 83, 56, 6, 50, 81, 85, 40, 9, 60, 59, 62, 15, 12, 32, 66, 47, 18, 19, 1, 37, 19, 47, 30, 44, 50, 21, 38, 42, 63, 56, 44, 79, 166, 340, 95, 99, 55, 47, 58, 10, 29, 8, 25, 43, 64, 44, 50, 28, 45, 35, 11, 41, 45, 31, 36, 76, 72, 60, 24, 30, 43, 7, 28, 25, 47, 31, 31, 36, 37, 37, 98, 94, 59, 41, 49, 22, 58, 68, 67, 80, 32, 29, 36, 73, 64, 57, 50, 43, 32, 18, 27, 48, 94, 100, 32, 37, 68, 182, 94, 63, 68, 67, 26, 46, 32, 48, 37, 43, 81, 15, 48, 40, 73, 65, 50, 67, 56, 4, 40, 40, 18, 70, 15, 14, 37, 21, 25, 55, 35, 19, 46, 11, 88, 42, 56, 54, 41, 30, 85, 12, 53, 68, 54, 89, 58, 52, 31, 30, 56, 39, 44, 13]], "x": ["2017-06-16", "2017-06-17", "2017-06-18", "2017-06-19", "2017-06-20", "2017-06-21", "2017-06-22", "2017-06-23", "2017-06-24", "2017-06-25", "2017-06-26", "2017-06-27", "2017-06-28", "2017-06-29", "2017-06-30", "2017-07-01", "2017-07-16", "2017-07-17", "2017-07-18", "2017-07-19", "2017-07-20", "2017-07-21", "2017-07-22", "2017-07-24", "2017-07-25", "2017-07-26", "2017-07-27", "2017-07-28", "2017-07-29", "2017-07-30", "2017-07-31", "2017-08-01", "2017-08-02", "2017-08-03", "2017-08-04", "2017-08-05", "2017-08-06", "2017-08-07", "2017-08-08", "2017-08-09", "2017-08-10", "2017-08-11", "2017-08-12", "2017-08-13", "2017-08-14", "2017-08-15", "2017-08-16", "2017-08-17", "2017-08-18", "2017-08-19", "2017-08-21", "2017-08-22", "2017-08-23", "2017-08-24", "2017-08-25", "2017-08-26", "2017-08-27", "2017-08-28", "2017-08-29", "2017-08-30", "2017-08-31", "2017-09-01", "2017-09-02", "2017-09-03", "2017-09-04", "2017-09-05", "2017-09-06", "2017-09-07", "2017-09-08", "2017-09-09", "2017-09-10", "2017-09-11", "2017-09-12", "2017-09-13", "2017-09-14", "2017-09-15", "2017-09-16", "2017-09-17", "2017-09-18", "2017-09-19", "2017-09-20", "2017-09-21", "2017-09-22", "2017-09-23", "2017-09-24", "2017-09-25", "2017-09-26", "2017-09-27", "2017-09-28", "2017-09-29", "2017-09-30", "2017-10-01", "2017-10-02", "2017-10-03", "2017-10-04", "2017-10-05", "2017-10-06", "2017-10-07", "2017-10-08", "2017-10-10", "2017-10-11", "2017-10-12", "2017-10-13", "2017-10-14", "2017-10-15", "2017-10-16", "2017-10-17", "2017-10-18", "2017-10-19", "2017-10-20", "2017-10-21", "2017-10-22", "2017-10-23", "2017-10-24", "2017-10-25", "2017-10-26", "2017-10-27", "2017-10-28", "2017-10-29", "2017-10-30", "2017-10-31", "2017-11-01", "2017-11-02", "2017-11-03", "2017-11-04", "2017-11-05", "2017-11-06", "2017-11-07", "2017-11-08", "2017-11-09", "2017-11-10", "2017-11-11", "2017-11-12", "2017-11-13", "2017-11-14", "2017-11-15", "2017-11-16", "2017-11-17", "2017-11-18", "2017-11-19", "2017-11-20", "2017-11-21", "2017-11-22", "2017-11-23", "2017-11-24", "2017-11-25", "2017-11-26", "2017-11-27", "2017-11-28", "2017-11-29", "2017-11-30", "2017-12-01", "2017-12-02", "2017-12-03", "2017-12-04", "2017-12-05", "2017-12-06", "2017-12-07", "2017-12-08", "2017-12-09", "2017-12-10", "2017-12-11", "2017-12-12", "2017-12-13", "2017-12-14", "2017-12-15", "2017-12-16", "2017-12-17", "2017-12-18", "2017-12-19", "2017-12-20", "2017-12-21", "2017-12-22", "2017-12-23", "2017-12-24", "2017-12-25", "2017-12-26", "2017-12-27", "2017-12-28", "2017-12-29", "2017-12-30", "2017-12-31", "2018-01-01", "2018-01-02", "2018-01-03", "2018-01-04", "2018-01-05", "2018-01-06", "2018-01-07", "2018-01-08", "2018-01-09", "2018-01-10", "2018-01-11", "2018-01-12", "2018-01-13", "2018-01-14", "2018-01-15", "2018-01-16", "2018-01-17", "2018-01-18", "2018-01-19", "2018-01-20", "2018-01-21", "2018-01-22", "2018-01-23", "2018-01-24", "2018-01-25", "2018-01-26", "2018-01-27", "2018-01-28", "2018-01-29", "2018-01-30", "2018-01-31", "2018-02-01", "2018-02-02", "2018-02-03", "2018-02-04", "2018-02-05", "2018-02-06", "2018-02-07", "2018-02-08", "2018-02-09", "2018-02-10", "2018-02-11", "2018-02-12", "2018-02-13", "2018-02-14", "2018-02-15", "2018-02-16", "2018-02-17", "2018-02-18", "2018-02-19", "2018-02-20", "2018-02-21", "2018-02-22", "2018-02-23", "2018-02-24", "2018-02-25", "2018-02-26", "2018-02-27", "2018-02-28", "2018-03-01", "2018-03-02", "2018-03-03", "2018-03-04", "2018-03-05", "2018-03-06", "2018-03-07", "2018-03-08", "2018-03-09", "2018-03-10", "2018-03-11", "2018-03-12", "2018-03-13", "2018-03-14", "2018-03-15", "2018-03-16", "2018-03-17", "2018-03-18", "2018-03-19", "2018-03-20", "2018-03-21", "2018-03-22", "2018-03-23", "2018-03-24", "2018-03-25", "2018-03-26", "2018-03-27", "2018-03-28", "2018-03-29", "2018-03-30", "2018-03-31", "2018-04-01", "2018-04-02", "2018-04-03", "2018-04-04", "2018-04-05", "2018-04-06", "2018-04-07", "2018-04-08", "2018-04-09", "2018-04-10", "2018-04-11", "2018-04-12", "2018-04-15", "2018-04-16", "2018-04-17", "2018-04-18", "2018-04-19", "2018-04-20", "2018-04-21", "2018-04-22", "2018-04-23"], "y": ["Adam", "Bridget"], "colorscale": " Viridis", "ygap": 0.5}], {"title": "Messages by Over Time", "xaxis": {"title": "Date", "autorange": true, "showgrid": true, "zeroline": true, "showline": false, "autotick": true, "nticks": 18, "showticklabels": true}, "yaxis": {"title": "Person", "autorange": true, "showgrid": true, "zeroline": true, "showline": true, "autotick": true, "ticks": "", "showticklabels": true}}, {"showLink": false, "linkText": "Export to plot.ly", "displayModeBar": false})});</script>



```python

```
