## Modifying ZIM Files

#### The Larger Picture
* Kiwix scrapes many useful sources, but sometimes the chunks are too big for IIAB.
* Using the zimdump program, the highly compressed ZIM files can be flattened into a file tree, modified, and then re-packaged as a ZIM file.
* This Notebook has a collection of tools which help in the above process.


#### How to Use this notebook
* There are install steps that only need to happen once. The cells containing these steps are set to "Raw" in the right most dropdown so that they do not execute automatically each time the notebook starts.
* The following bash script successfully installed zimtools on Ubuntu 20.04.It only needs to be run once. I think it's easier to do it from the command line, with tab completion. In a terminal, do the following:

```
cd /opt/iiab/iiab-factory/content/kiwix/generic/ 
sudo ./install-zim-tools.sh
```

* **Some conventions**: Jupyter does not want to run as root. We will create a file structure that exists in the users home directory -- so the application will be able to write all the files it needs to function.
```
<home directory>
├── new_zim
├── tree
└── working
```
In general terms, this program will dump the zim data into "tree", modify it, gather additional data into "working"
, and create a ZIM file in "new_zim"
* For testing purposes, the user will need to link from the server's document root to her home directory (so that the nginx http server in IIAB will serve the candidate in "tree):

```
cd
mkdir -p zimtest
ln -s /home/<user name>/zimtest /library/www/html/zimtest 
```


#### Installation Notes to myself
* Installing on Windows 10 insider preview WSL2. Used https://towardsdatascience.com/configuring-jupyter-notebook-in-windows-subsystem-linux-wsl2-c757893e9d69.
* First tried installing miniconda, and then installing jupyterlab with it.
* Wanted VIM bindings to edit cells, but jupyterlab version insralled by conda was too old for jupyter-vim extenion. Wound up deleting old version with conda, and used pip to install both.
* Jupyterlab seems to make the current directory its root. I created a notebook directory, and aways start jupyter lab from my home directiry
* Discovered that I could enable writing by non-root group in the iiab-factory repo, and continue to use git for version control. Needed to make symbolic link from ~/miniconda to iiab-factory.
* Reminder: Start jupyterlav in console via "jupyter lab --no-browser", and then pasteing the html link displayed into my browser.

#### Declare input and output
* The ZIM file names tend to be long and hard to remember. The PROJECT_NAME, initialized below, is used to create path names. All of the output of the zimdump program is placed in \<home\>/zimtest/\<PROJECT_NAME\>/tree. All if the intermediate downloads, and data, are placed in \<home\>/zimtest/\<PROJECT_NAME\>/working. If you use the IIAB Admin Console to download ZIMS, you will find them in /library/zims/content/.


```python
# -*- coding: utf-8 -*-
from __future__ import unicode_literals
import os,sys
import json
import youtube_dl
import pprint as pprint

# Declare a short project name (ZIM files are often long strings
#PROJECT_NAME = 'ted-kiwix'
PROJECT_NAME = 'ted-kiwix'
# Input the full path of the downloaded ZIM file
ZIM_PATH = '/home/ghunt/zimtest/ted-kiwix/zim-src/teded_en_all_2021-01.zim'

# The rest of the paths are computed and represent the standard layout
# Jupyter sets a working director as part of it's setup. We need it's value
HOME = os.environ['HOME']
WORKING_DIR = HOME + '/zimtest/' + PROJECT_NAME + '/working'
PROJECT_DIR = HOME + '/zimtest/' + PROJECT_NAME + '/tree'
OUTPUT_DIR = HOME + '/zimtest/' + PROJECT_NAME + '/new-zim'
SOURCE_DIR = HOME + '/zimtest/' + PROJECT_NAME + '/zim-src'
dir_list = ['new-zim','tree','working/video_json','zim-src']
for f in dir_list: 
    if not os.path.isdir(HOME + '/zimtest/' + PROJECT_NAME +'/' + f):
       os.makedirs(HOME + '/zimtest/' + PROJECT_NAME +'/' + f)

# abort if the input file cannot be found
if not os.path.exists(ZIM_PATH):
    print('%s path not found. Quitting. . .'%ZIM_PATH)
    exit

```


```python
# First we need to get a current copy of the script
dest = HOME + '/zimtest'
%cp /opt/iiab/iiab-factory/content/kiwix/de-namespace.sh {dest} 
```


```python
# The following command will zimdump to the "tree" directory
#   and remove the namespace directories
# It will return without doing anything if the "tree' is not empty
progname = HOME + '/zimtest/de-namespace.sh'
!{progname} {ZIM_PATH} {PROJECT_NAME}
```

    + set -e
    + '[' 2 -lt 2 ']'
    + '[' '!' -f /home/ghunt/zimtest/ted-kiwix/zim-src/teded_en_all_2021-01.zim ']'
    ++ ls /home/ghunt/zimtest/ted-kiwix/tree
    ++ wc -l
    + contents=0
    + '[' 0 -ne 0 ']'
    + rm -rf /home/ghunt/zimtest/ted-kiwix/tree
    + mkdir -p /home/ghunt/zimtest/ted-kiwix/tree
    + echo 'This de-namespace file reminds you that this folder will be overwritten?'
    + zimdump dump --dir=/home/ghunt/zimtest/ted-kiwix/tree /home/ghunt/zimtest/ted-kiwix/zim-src/teded_en_all_2021-01.zim
    + exit 0


* The next step is a manual one that you will need to do with your browser. That is: to verify that after the namespace directories were removed, and all of the html links have been adjusted correctly. Point your browser to <hostname>/zimtest/\<PROJECT_NAME\>/tree.
* If everything is working, it's time to go fetch the information about each video from youtube.


```python
ydl = youtube_dl.YoutubeDL()

downloaded = 0
skipped = 0
# Create a list of youtube id's
yt_id_list = os.listdir(PROJECT_DIR + '/I/videos/')
for yt_id in iter(yt_id_list):
    if os.path.exists(WORKING_DIR + '/video_json/' + yt_id + '.json'):
        # skip over items that are already downloadd
        skipped += 1
        continue
    with ydl:
       result = ydl.extract_info(
                'http://www.youtube.com/watch?v=%s'%yt_id,
                download=False # We just want to extract the info
                )
       downloaded += 1

    with open(WORKING_DIR + '/video_json/' + yt_id + '.json','w') as fp:
        fp.write(json.dumps(result))
    #pprint.pprint(result['upload_date'],result['view_count'])
print('%s skipped and %s downloaded'%(skipped,downloaded))
```

#### Playlist Navigation to Videos
* On the home page there is a drop down selector which lists about 70 cateegories (or playlists).
* The value from that drop down is used to pick an entry in "-/assets/data.js", which in turn specifies the playlist of yourtube ID"s that are displayed when a selection is made.


```python
def get_assets_data():
    # the file <root>/assets/data.js holds the category to video mappings
    outstr = ''
    data = {}
    with open(PROJECT_DIR + '/-/assets/data.js', 'r') as fp:
    #with open(OUTPUT_DIR + '/I/assets/data.js', 'r') as fp:
        while True:
            line = fp.readline()
            if not line:
                break
            if line.startswith('var'):
                if len(outstr) > 1:
                    # clip off the trailing semicolon
                    outstr = outstr[:-2]
                    try:
                        data[cat] = json.loads(outstr)
                    except Exception:
                        print('Parse error: %s'%outstr[:80])
                        exit
                cat = line[9:-4]
                outstr = '['
            else:
                outstr += line
    return data

zim_category_js = get_assets_data()

def get_zim_data(yt_id):
    rtn_dict = {}
    for cat in zim_category_js:
        for video in range(len(zim_category_js[cat])):
            if zim_category_js[cat][video]['id'] == yt_id:
                rtn_dict = zim_category_js[cat][video]
                break
        if len(rtn_dict) > 0: break
    return rtn_dict
```


```python
# enable this cell if you want summarize each category, and get a total of videos
#   including those that are in more than one categofy
tot=0
for cat in zim_category_js:
    tot += len(zim_category_js[cat])
    print(cat, len(zim_category_js[cat]))
print('Number of Videos in all categories -- perhaps used more than once:%d'%tot)
```

    the_power_of_nature  7
    the_basics_of_quantum_mechanics  11
    ted_ed_loves_trees  8
    mind_matters  48
    student_voices_from_tedtalksed  3
    ecofying_cities  13
    tedyouth_talks  41
    the_artist_s_palette  20
    creative_writing_workshop_campyoutube_withme  11
    even_more_ted_ed_originals  160
    new_ted_ed_originals  948
    our_changing_climate  43
    the_writer_s_workshop  27
    can_you_solve_this_riddle  56
    ted_ed_riddles_season_3  7
    moments_of_vision  12
    hone_your_media_literacy_skills  11
    ted_ed_riddles_season_1  8
    making_the_invisible_visible  9
    visualizing_data  11
    math_in_real_life  85
    exploring_the_senses  10
    out_of_this_world  43
    government_declassified  37
    think_like_a_coder_campyoutube_withme  13
    reading_between_the_lines  55
    the_world_s_people_and_places  134
    the_works_of_william_shakespeare  9
    the_world_of_sports  12
    elections_in_the_united_states  9
    awesome_nature  128
    the_great_thanksgiving_car_ride  8
    the_wonders_of_earth  13
    humans_vs_viruses  18
    before_and_after_einstein  47
    understanding_genetics  12
    how_things_work  70
    cern_space_time_101  3
    questions_no_one_yet_knows_the_answers_to  8
    ted_ed_riddles_season_2  8
    cyber_influence_power  29
    playing_with_language  26
    inventions_that_shaped_history  48
    facing_our_ugly_history  3
    the_big_questions  30
    think_like_a_coder  11
    love_actually  6
    there_s_a_poem_for_that_season_1  12
    mysteries_of_vernacular  26
    behind_the_curtain  30
    troubleshooting_the_world  86
    national_teacher_day  19
    myths_from_around_the_world  35
    superhero_science  7
    ted_ed_professional_development  5
    discovering_the_deep  28
    more_money_more_problems  9
    well_behaved_women_seldom_make_history  37
    getting_under_our_skin  163
    math_of_the_impossible  13
    history_vs  11
    brain_discoveries  13
    more_book_recommendations_from_ted_ed  38
    more_ted_ed_originals  174
    integrated_photonics  3
    a_day_in_the_life  14
    the_way_we_think  50
    ted_ed_weekend_student_talks  26
    animation_basics  12
    you_are_what_you_eat  15
    ted_ed_riddles_season_4  9
    uploads_from_ted_ed  1763
    actions_and_reactions  48
    Number of Videos in all categories -- perhaps used more than once:4975


#### The following Cell is subroutines and can be left minimized


```python
from pprint import pprint
from pymediainfo import MediaInfo

def mediainfo_dict(path):
    try:
        minfo = MediaInfo.parse(path)
    except:
        print('mediainfo_dict. file not found: %s'%path)
        return {}
    return minfo.to_data()

def select_info(path):
    global data
    data = mediainfo_dict(path)
    rtn = {}
    for index in range(len(data['tracks'])):
        track = data['tracks'][index]
        if track['kind_of_stream'] == 'General':
            rtn['file_size'] = track.get('file_size',0)
            rtn['bit_rate'] = track.get('overall_bit_rate',0)
            rtn['time'] = track['other_duration'][0]
        if track['kind_of_stream'] == 'Audio':
            rtn['a_stream'] = track.get('stream_size',0)
            rtn['a_rate'] = track.get('maximum_bit_rate',0)
            rtn['a_channels'] = track.get('channel_s',0)
        if track['kind_of_stream'] == 'Video':
            rtn['v_stream'] = track.get('stream_size',0)
            rtn['v_format'] = track['other_format'][0]
            rtn['v_rate'] = track.get('bit_rate',0)
            rtn['v_frame_rate'] = track.get('frame_rate',0)
            rtn['v_width'] = track.get('width',0)
            rtn['v_height'] = track.get('height',0)
    return rtn
```


```python
import sqlite3
class Sqlite():
   def __init__(self, filename):
      self.conn = sqlite3.connect(filename)
      self.conn.row_factory = sqlite3.Row
      self.conn.text_factory = str
      self.c = self.conn.cursor()
    
   def __del__(self):
      self.conn.commit()
      self.c.close()
      del self.conn

def get_video_json(path):
    with open(path,'r') as fp:
        try:
            jsonstr = fp.read()
            #print(path)
            modules = json.loads(jsonstr.strip())
        except Exception as e:
            print(e)
            print(jsonstr[:80])
            return {}
    return modules

def video_size(yt_id):
    return os.path.getsize(PROJECT_DIR + '/-/videos/' + yt_id + '/video.webm')

def make_directory(path):
    if not os.path.exists(path):
        os.makedirs(path)

def download_file(url,todir):
    local_filename = url.split('/')[-1]
    r = requests.get(url)
    f = open(todir + '/' + local_filename, 'wb')
    for chunk in r.iter_content(chunk_size=512 * 1024):
        if chunk:
            f.write(chunk)
    f.close()
    
from datetime import datetime
def age_in_years(upload_date):
    uploaded_dt = datetime.strptime(upload_date,"%Y%m%d")
    now_dt = datetime.now()
    days_delta = now_dt - uploaded_dt
    years = days_delta.days/365 + 1
    return years
```

#### Create a sqlite database which collects Data about each Video
* We've already downloaded the data from YouTube for each Video. So get the items that are interesing to us. Such as size,date uploaded to youtube,view count


```python
def initialize_db():
    sql = 'CREATE TABLE IF NOT EXISTS video_info ('\
            'yt_id TEXT UNIQUE, zim_size INTEGER, view_count INTEGER, age INTEGER, '\
            'views_per_year INTEGER, upload_date TEXT, duration TEXT, '\
            'height INTEGER, width INTEGER,'\
            'bit_rate TEXT, format TEXT, '\
            'average_rating REAL,slug TEXT)'
    db.c.execute(sql)
    
db = Sqlite(WORKING_DIR + '/zim_video_info.sqlite')
initialize_db()
for yt_id in iter(yt_id_list):
    
    # fetch data from assets/data.js
    zim_data = get_zim_data(yt_id)
    #if len(zim_data) == 0: continue
    slug = zim_data['slug']
    # We already have youtube data for every video, use it 
    data = get_video_json(WORKING_DIR + "/video_json/" + yt_id + '.json')
    #if len(data) == 0:continue
    vsize = data.get('filesize',0)
    view_count = data['view_count']
    upload_date = data['upload_date']
    average_rating = data['average_rating']
    # calculate the views_per_year since it was uploaded
    age = round(age_in_years(upload_date))
    views_per_year = int(view_count / age)
        
    # interogate the video itself
    filename = PROJECT_DIR + '/I/videos/' + yt_id + '/video.webm'
    vsize = os.path.getsize(filename)
    selected_data = select_info(filename)
    if len(selected_data) == 0:
        duration = "not found"
        bit_rate = "" 
        v_format = ""
    else:
        duration = selected_data['time']
        bit_rate = selected_data['bit_rate']
        v_format = selected_data['v_format']
        v_height = selected_data['v_height']
        v_width = selected_data['v_width']
    
    # colums names: yt_id,zim_size,view_count,views_per_year,upload_date,duration,
    #         bit_rate, format,average_rating,slug
    sql = 'INSERT OR REPLACE INTO video_info VALUES (?,?,?,?,?,?,?,?,?,?,?,?,?)'
    db.c.execute(sql,[yt_id,vsize,view_count,round(age),views_per_year,upload_date, \
                      duration,v_height,v_width,bit_rate,v_format,average_rating,slug, ])
db.conn.commit()
```


```python
print(yt_id,vsize,view_count,views_per_year,upload_date, \
                      duration,bit_rate,v_format,average_rating,slug,round(age))
```

    I2apGYUX7Q0 16108773 2872534 287253 20120311  354180  4.7892714 why_can_t_we_see_evidence_of_alien_life 10



```python
sqlite_db = WORKING_DIR + '/zim_video_info.sqlite'
!sqlite3 {sqlite_db} '.headers on' 'select * from video_info limit 2'
```

    yt_id|zim_size|view_count|age|views_per_year|upload_date|duration|height|width|bit_rate|format|average_rating|slug
    9mcuIc5O-DE|8552838|3212048|10|321204|20120626|4 min 13 s|270|480|270120|VP8|4.9190717|how_do_pain_relievers_work_george_zaidan
    XSb-pIloOFc|5605385|34034|9|3781|20130127|3 min 50 s|270|480|194888|VP8|4.9685745|the_coming_neurological_epidemic_gregory_petsko


#### Select the cutoff using view count and total size
* Order the videos by view count. Then select the sum line in the that has the target sum.


```python
def human_readable(num):
    # return 3 significant digits and unit specifier
    num = float(num)
    units = [ '','K','M','G']
    for i in range(4):
        if num<10.0:
            return "%.2f%s"%(num,units[i])
        if num<100.0:
            return "%.1f%s"%(num,units[i])
        if num < 1000.0:
            return "%.0f%s"%(num,units[i])
        num /= 1024.0

sql = 'select slug,zim_size,views_per_year,view_count,duration,upload_date,'\
       'format,width,height,bit_rate from video_info order by views_per_year desc'
tot_sum = 0
db.c.execute(sql)
rows = db.c.fetchall()
print('%60s %6s %6s %6s %6s %8s %8s'%('Name','Size','Sum','Views','Views','Date  ','Duration'))
print('%60s %6s %6s %6s %6s %8s %8s'%('','','','','/ yr','',''))
for row in rows:
    tot_sum += row['zim_size']
    print('%60s %6s %6s %6s %6s %8s %8s'%(row['slug'][:60],human_readable(row['zim_size']),\
                              human_readable(tot_sum),human_readable(row['view_count']),\
                              human_readable(row['views_per_year']),\
                              row['upload_date'],row['duration']))
```

                                                            Name   Size    Sum  Views  Views   Date   Duration
                                                                                        / yr                  
              can_you_solve_the_prisoner_hat_riddle_alex_gendler  6.65M  6.65M  19.6M  3.27M 20151005 4 min 34 s
                          how_thor_got_his_hammer_scott_a_mellor  8.02M  14.7M  9.70M  3.23M 20190107 4 min 51 s
                    which_is_stronger_glue_or_tape_elizabeth_cox  12.6M  27.3M  10.9M  2.71M 20180430 4 min 50 s
         what_are_those_floaty_things_in_your_eye_michael_mauser  6.76M  34.0M  18.6M  2.66M 20141201 4 min 4 s
                     is_marijuana_bad_for_your_brain_anees_bahji  12.1M  46.2M  4.90M  2.45M 20191202 6 min 43 s
                        the_infinite_hotel_paradox_jeff_dekofsky  9.92M  56.1M  19.2M  2.40M 20140116 5 min 59 s
                    can_you_solve_the_bridge_riddle_alex_gendler  6.09M  62.2M  16.7M  2.38M 20150901 3 min 49 s
                       a_brie_f_history_of_cheese_paul_kindstedt  7.95M  70.1M  7.08M  2.36M 20181213 5 min 33 s
     why_don_t_perpetual_motion_machines_ever_work_netta_schramm  16.6M  86.7M  11.0M  2.21M 20170605 5 min 30 s
             can_you_solve_the_wizard_standoff_riddle_dan_finkel  8.15M  94.9M  8.77M  2.19M 20180522 5 min 25 s
                             history_s_worst_nun_theresa_a_yugar  13.6M   109M  4.32M  2.16M 20191121 4 min 46 s
              questions_no_one_knows_the_answers_to_full_version  34.0M   143M  21.5M  2.15M 20120317 12 min 7 s
                           a_brief_history_of_chess_alex_gendler  9.53M   152M  4.27M  2.14M 20190912 5 min 39 s
        the_tale_of_the_doctor_who_defied_death_iseult_gillespie  9.68M   162M  4.25M  2.12M 20200312 5 min 27 s
                               the_language_of_lying_noah_zandan  8.93M   171M  14.5M  2.07M 20141103 5 min 41 s
                          what_makes_muscles_grow_jeffrey_siegel  6.63M   177M  11.9M  1.99M 20151103 4 min 19 s
                how_do_cigarettes_affect_the_body_krishna_sudhir  12.7M   190M  5.87M  1.96M 20180913 5 min 20 s
                can_you_solve_the_three_gods_riddle_alex_gendler  7.41M   197M  9.53M  1.91M 20170221 4 min 53 s
             a_day_in_the_life_of_a_roman_soldier_robert_garland  9.16M   207M  7.60M  1.90M 20180329 4 min 59 s
                             the_surprising_effects_of_pregnancy  19.8M   226M  1.72M  1.72M 20201001 5 min 45 s
                       can_you_solve_the_virus_riddle_lisa_winer  9.66M   236M  8.59M  1.72M 20170403 5 min 12 s
               can_you_solve_einsteins_riddle_dan_van_der_vieren  8.86M   245M  10.3M  1.71M 20151130 5 min 12 s
    how_to_practice_effectively_for_just_about_anything_annie_bo  9.17M   254M  8.51M  1.70M 20170227 4 min 49 s
      why_incompetent_people_think_they_re_amazing_david_dunning  10.9M   265M  6.66M  1.67M 20171109 5 min 7 s
             can_you_solve_the_prisoner_boxes_riddle_yossi_elran  9.44M   274M  8.27M  1.65M 20161003 4 min 51 s
    can_you_solve_the_famously_difficult_green_eyed_logic_puzzle  8.05M   282M  11.5M  1.64M 20150616 4 min 41 s
                            the_myth_of_arachne_iseult_gillespie  9.15M   292M  6.47M  1.62M 20180208 4 min 29 s
                       the_myth_of_pandoras_box_iseult_gillespie  10.9M   303M  4.74M  1.58M 20190115 4 min 9 s
        what_would_happen_if_you_didnt_drink_water_mia_nacamulli  8.75M   311M  9.41M  1.57M 20160329 4 min 51 s
            what_would_happen_if_you_didnt_sleep_claudia_aguirre  7.82M   319M  9.25M  1.54M 20151112 4 min 34 s
                      can_you_solve_the_locker_riddle_lisa_winer  6.77M   326M  9.20M  1.53M 20160328 3 min 49 s
                               the_myth_of_sisyphus_alex_gendler  9.76M   336M  4.49M  1.50M 20181113 4 min 56 s
                    can_you_solve_the_pirate_riddle_alex_gendler  9.00M   345M  7.47M  1.49M 20170501 5 min 23 s
    what_really_happened_during_the_salem_witch_trials_brian_a_p  10.1M   355M  2.94M  1.47M 20200504 5 min 30 s
                      how_your_digestive_system_works_emma_bryce  6.85M   362M  5.87M  1.47M 20171214 4 min 56 s
                     the_myth_of_cupid_and_psyche_brendan_pelsue  16.4M   378M  7.31M  1.46M 20170803 5 min 32 s
    which_is_better_soap_or_hand_sanitizer_alex_rosenthal_and_pa  14.0M   392M  2.90M  1.45M 20200505 6 min 14 s
                      the_loathsome_lethal_mosquito_rose_eveleth  4.87M   397M  11.6M  1.45M 20131202 2 min 39 s
                          does_time_exist_andrew_zimmerman_jones  15.1M   412M  4.34M  1.45M 20181023 5 min 15 s
           how_the_food_you_eat_affects_your_brain_mia_nacamulli  14.6M   427M  8.66M  1.44M 20160621 4 min 52 s
           the_myth_of_hercules_12_labors_in_8_bits_alex_gendler  23.8M   450M  4.31M  1.44M 20180925 7 min 50 s
            can_you_solve_the_unstoppable_blob_riddle_dan_finkel  5.70M   456M  4.29M  1.43M 20190318 3 min 42 s
    the_myth_of_thor_s_journey_to_the_land_of_giants_scott_a_mel  7.63M   464M  5.70M  1.42M 20180222 5 min 17 s
                        why_do_cats_act_so_weird_tony_buffington  11.9M   476M  8.43M  1.40M 20160426 4 min 57 s
                                         when_is_a_pandemic_over  15.3M   491M  2.81M  1.40M 20200601 5 min 52 s
                         the_myth_of_prometheus_iseult_gillespie  8.23M   499M  5.58M  1.39M 20171114 4 min 46 s
                             why_can_t_you_divide_by_zero_ted_ed  7.70M   507M  5.58M  1.39M 20180423 4 min 50 s
                         a_brief_history_of_alcohol_rod_phillips  8.92M   516M  2.77M  1.38M 20200102 5 min 20 s
    who_were_the_vestal_virgins_and_what_was_their_job_peta_gree  12.8M   529M  6.87M  1.37M 20170530 4 min 32 s
                  einstein_s_twin_paradox_explained_amber_stuver  11.7M   540M  2.73M  1.36M 20190926 6 min 15 s
          the_tragic_myth_of_orpheus_and_eurydice_brendan_pelsue  7.92M   548M  5.44M  1.36M 20180111 4 min 41 s
                 can_you_solve_the_temple_riddle_dennis_e_shasha  6.70M   555M  8.13M  1.36M 20160201 4 min 12 s
                   can_you_solve_the_egg_drop_riddle_yossi_elran  8.41M   563M  5.39M  1.35M 20171107 4 min 46 s
           can_you_solve_the_counterfeit_coin_riddle_jennifer_lu  9.45M   573M  6.67M  1.33M 20170103 4 min 34 s
    what_happens_to_our_bodies_after_we_die_farnaz_khatibi_jafar  8.41M   581M  6.67M  1.33M 20161013 4 min 40 s
      the_tale_of_the_boy_who_tricked_the_devil_iseult_gillespie  17.9M   599M  2.66M  1.33M 20200707 5 min 25 s
        the_chinese_myth_of_the_immortal_white_snake_shunan_teng  7.80M   607M  3.97M  1.32M 20190507 4 min 0 s
                     how_does_alcohol_make_you_drunk_judy_grisel  8.19M   615M  2.63M  1.31M 20200409 5 min 25 s
                           how_does_anesthesia_work_steven_zheng  8.05M   623M  7.83M  1.31M 20151207 4 min 55 s
                 the_benefits_of_a_bilingual_brain_mia_nacamulli  11.5M   635M  9.11M  1.30M 20150623 5 min 3 s
    the_myth_of_king_midas_and_his_golden_touch_iseult_gillespie  16.3M   651M  5.03M  1.26M 20180305 5 min 4 s
                    can_you_solve_the_passcode_riddle_ganesh_pai  7.16M   658M  7.48M  1.25M 20160707 4 min 7 s
          how_does_the_rorschach_inkblot_test_work_damion_searls  12.9M   671M  3.71M  1.24M 20190305 4 min 57 s
                        how_sugar_affects_the_brain_nicole_avena  16.5M   688M  9.87M  1.23M 20140107 5 min 2 s
     why_doesnt_the_leaning_tower_of_pisa_fall_over_alex_gendler  9.48M   697M  2.46M  1.23M 20191203 5 min 5 s
      the_history_of_the_world_according_to_cats_eva_maria_geigl  11.9M   709M  3.68M  1.23M 20190103 4 min 34 s
                   cannibalism_in_the_animal_kingdom_bill_schutt  9.64M   719M  4.90M  1.22M 20180319 4 min 57 s
                   the_psychology_of_narcissism_w_keith_campbell  8.95M   728M  7.34M  1.22M 20160223 5 min 9 s
    which_type_of_milk_is_best_for_you_jonathan_j_osullivan_grac  19.1M   747M  1.20M  1.20M 20201020 5 min 25 s
    mating_frenzies_sperm_hoards_and_brood_raids_the_life_of_a_f  12.0M   759M  2.35M  1.18M 20200116 5 min 18 s
     how_playing_an_instrument_benefits_your_brain_anita_collins  8.59M   767M  9.15M  1.14M 20140722 4 min 44 s
                 how_does_the_stock_market_work_oliver_elfenbaum  9.18M   776M  3.43M  1.14M 20190429 4 min 29 s
          the_rise_and_fall_of_the_berlin_wall_konrad_h_jarausch  9.52M   786M  5.71M  1.14M 20170816 6 min 25 s
                      how_does_a_jellyfish_sting_neosha_s_kashef  7.51M   793M  7.98M  1.14M 20150810 4 min 16 s
         5_tips_to_improve_your_critical_thinking_samantha_agoos  6.79M   800M  6.83M  1.14M 20160315 4 min 29 s
    from_slave_to_rebel_gladiator_the_life_of_spartacus_fiona_ra  19.2M   819M  3.41M  1.14M 20181217 5 min 15 s
                    the_benefits_of_good_posture_murat_dalkilinc  5.78M   825M  7.90M  1.13M 20150730 4 min 26 s
                              what_is_depression_helen_m_farrell  7.12M   832M  6.75M  1.13M 20151215 4 min 28 s
    what_if_there_were_1_trillion_more_trees_jean_francois_basti  24.6M   857M  1.12M  1.12M 20201027 5 min 43 s
    how_did_hitler_rise_to_power_alex_gendler_and_anthony_hazard  11.7M   869M  6.75M  1.12M 20160718 5 min 36 s
          a_glimpse_of_teenage_life_in_ancient_rome_ray_laurence  15.9M   884M  10.1M  1.12M 20121029 6 min 34 s
                     the_worlds_most_mysterious_book_stephen_bax  9.37M   894M  5.59M  1.12M 20170525 4 min 42 s
                          3_tips_to_boost_your_confidence_ted_ed  9.28M   903M  6.67M  1.11M 20151006 4 min 16 s
                      can_you_solve_the_frog_riddle_derek_abbott  6.91M   910M  6.61M  1.10M 20160229 4 min 30 s
    the_unexpected_math_behind_van_gogh_s_starry_night_natalya_s  14.7M   925M  7.70M  1.10M 20141030 4 min 38 s
                      the_myth_of_icarus_and_daedalus_amy_adkins  15.2M   940M  5.48M  1.10M 20170313 5 min 8 s
    a_day_in_the_life_of_an_ancient_egyptian_doctor_elizabeth_co  7.03M   947M  4.36M  1.09M 20180719 4 min 33 s
    the_myth_behind_the_chinese_zodiac_megan_campisi_and_pen_pen  6.53M   953M  5.44M  1.09M 20170126 4 min 22 s
    the_atlantic_slave_trade_what_too_few_textbooks_told_you_ant  10.8M   964M  7.56M  1.08M 20141222 5 min 38 s
                    the_philosophy_of_stoicism_massimo_pigliucci  10.3M   975M  5.39M  1.08M 20170619 5 min 29 s
                           historys_deadliest_colors_j_v_maranto  14.5M   989M  5.36M  1.07M 20170522 5 min 13 s
                      how_to_spot_a_pyramid_scheme_stacie_bosley  7.23M   996M  3.21M  1.07M 20190402 5 min 1 s
                             are_the_illuminati_real_chip_berlet  7.43M  0.98G  2.12M  1.06M 20191031 4 min 57 s
                               what_is_schizophrenia_anees_bahji  10.5M  0.99G  2.08M  1.04M 20200326 5 min 32 s
              can_you_solve_the_river_crossing_riddle_lisa_winer  8.29M  1.00G  5.17M  1.03M 20161101 4 min 18 s
                 can_you_solve_the_airplane_riddle_judd_a_schorr  9.30M  1.01G  5.10M  1.02M 20161201 4 min 37 s
                     the_history_of_chocolate_deanna_pucciarelli  8.67M  1.02G  5.08M  1.02M 20170316 4 min 40 s
            hawking_s_black_hole_paradox_explained_fabio_pacucci  11.2M  1.03G  2.01M  1.01M 20191022 5 min 37 s
    the_legend_of_annapurna_hindu_goddess_of_nourishment_antara_  8.40M  1.04G  1.98M  0.99M 20200213 5 min 8 s
                    can_you_solve_the_ragnarok_riddle_dan_finkel  22.4M  1.06G  1.97M  0.98M 20200630 5 min 14 s
                                       why_do_women_have_periods  7.05M  1.06G  5.88M  0.98M 20151019 4 min 45 s
                          the_bug_that_poops_candy_george_zaidan  14.1M  1.08G  1.96M  0.98M 20200414 4 min 46 s
    is_life_meaningless_and_other_absurd_questions_nina_medvinsk  18.8M  1.10G   994K   994K 20200921 6 min 12 s
                 why_are_some_people_left_handed_daniel_m_abrams  8.22M  1.10G  6.78M   992K 20150203 5 min 6 s
                                       why_do_we_itch_emma_bryce  13.6M  1.12G  4.83M   990K 20170411 4 min 43 s
        can_you_solve_the_penniless_pilgrim_riddle_daniel_finkel  6.60M  1.12G  3.85M   985K 20180604 4 min 44 s
       the_three_different_ways_mammals_give_birth_kate_slabosky  8.55M  1.13G  4.78M   979K 20170417 4 min 49 s
    is_human_evolution_speeding_up_or_slowing_down_laurence_hurs  21.1M  1.15G   979K   979K 20200922 5 min 47 s
      the_surprising_reason_our_muscles_get_tired_christian_moro  10.6M  1.16G  2.85M   972K 20190418 4 min 24 s
                     what_makes_something_kafkaesque_noah_tavlin  19.7M  1.18G  5.69M   971K 20160620 5 min 3 s
                    can_you_solve_the_fish_riddle_steve_wyborney  8.49M  1.19G  4.72M   966K 20170626 4 min 49 s
                   how_does_laser_eye_surgery_work_dan_reinstein  12.1M  1.20G  1.88M   963K 20191119 5 min 36 s
    what_would_happen_if_every_human_suddenly_disappeared_dan_kw  9.30M  1.21G  2.77M   947K 20180917 5 min 21 s
    the_egyptian_book_of_the_dead_a_guidebook_for_the_underworld  7.39M  1.22G  4.62M   947K 20161031 4 min 31 s
                               history_vs_cleopatra_alex_gendler  7.53M  1.23G  4.61M   945K 20170202 4 min 27 s
                   the_five_major_world_religions_john_bellaimey  36.6M  1.26G  7.33M   938K 20131114 11 min 9 s
                               how_do_tornadoes_form_james_spann  8.52M  1.27G  7.32M   937K 20140819 4 min 11 s
           history_through_the_eyes_of_a_chicken_chris_a_kniesly  14.1M  1.28G  2.73M   933K 20181011 5 min 11 s
          the_greek_myth_of_talos_the_first_robot_adrienne_mayor  8.36M  1.29G  1.79M   917K 20191024 4 min 6 s
                       four_sisters_in_ancient_rome_ray_laurence  17.2M  1.31G  8.05M   916K 20130514 8 min 38 s
                         the_dark_history_of_bananas_john_soluri  17.6M  1.33G   911K   911K 20201102 6 min 2 s
    how_to_manage_your_time_more_effectively_according_to_machin  7.65M  1.33G  3.55M   909K 20180102 5 min 9 s
                          where_does_gold_come_from_david_lunney  9.66M  1.34G  5.23M   893K 20151008 4 min 34 s
                      a_brief_history_of_cannibalism_bill_schutt  8.18M  1.35G  2.61M   892K 20190725 4 min 49 s
       the_rise_and_fall_of_the_byzantine_empire_leonora_neville  8.15M  1.36G  3.48M   890K 20180409 5 min 20 s
                                      why_do_we_dream_amy_adkins  13.5M  1.37G  5.14M   878K 20151210 5 min 37 s
             the_fascinating_history_of_cemeteries_keith_eggener  9.83M  1.38G  2.57M   878K 20181030 5 min 18 s
               what_happens_during_a_heart_attack_krishna_sudhir  8.09M  1.39G  4.27M   876K 20170214 4 min 53 s
                      why_sitting_is_bad_for_you_murat_dalkilinc  8.20M  1.40G  5.87M   859K 20150305 5 min 4 s
          why_should_you_read_dune_by_frank_herbert_dan_kwartler  12.1M  1.41G  1.68M   859K 20191217 5 min 5 s
                             history_vs_che_guevara_alex_gendler  11.8M  1.42G  3.35M   858K 20171127 6 min 7 s
    how_to_stay_calm_under_pressure_noa_kageyama_and_pen_pen_che  8.06M  1.43G  3.34M   854K 20180521 4 min 28 s
              can_you_solve_the_secret_sauce_riddle_alex_gendler  7.29M  1.44G  1.66M   850K 20190916 4 min 42 s
                   how_does_money_laundering_work_delena_d_spann  11.9M  1.45G  4.10M   840K 20170523 4 min 46 s
      the_great_conspiracy_against_julius_caesar_kathryn_tempest  10.4M  1.46G  5.73M   838K 20141218 5 min 57 s
    mansa_musa_one_of_the_wealthiest_people_who_ever_lived_jessi  12.4M  1.47G  5.73M   838K 20150518 3 min 54 s
    why_do_animals_have_such_different_lifespans_joao_pedro_de_m  7.36M  1.48G  4.02M   824K 20170404 4 min 56 s
                                  how_tsunamis_work_alex_gendler  8.95M  1.49G  6.42M   822K 20140424 3 min 36 s
       the_rise_and_fall_of_the_mongol_empire_anne_f_broadbridge  12.1M  1.50G  2.40M   820K 20190829 5 min 0 s
                                  the_history_of_tea_shunan_teng  7.57M  1.50G  3.98M   816K 20170516 4 min 57 s
            the_mysterious_life_and_death_of_rasputin_eden_girma  8.68M  1.51G  1.59M   816K 20200107 5 min 13 s
    licking_bees_and_pulping_trees_the_reign_of_a_wasp_queen_ken  10.9M  1.52G  1.59M   812K 20200128 5 min 22 s
                             why_are_sloths_so_slow_kenny_coogan  7.71M  1.53G  3.95M   809K 20170425 5 min 14 s
            can_you_solve_the_stolen_rubies_riddle_dennis_shasha  10.1M  1.54G  2.36M   807K 20181025 4 min 50 s
    why_isn_t_the_world_covered_in_poop_eleanor_slade_and_paul_m  7.34M  1.55G  3.10M   794K 20180326 4 min 57 s
                               what_makes_a_hero_matthew_winkler  11.4M  1.56G  6.94M   790K 20121204 4 min 33 s
    the_cambodian_myth_of_lightning_thunder_and_rain_prumsodun_o  11.6M  1.57G  3.07M   787K 20180410 4 min 37 s
                          how_does_asthma_work_christopher_e_gaw  8.91M  1.58G  3.84M   786K 20170511 5 min 9 s
    meet_the_tardigrade_the_toughest_animal_on_earth_thomas_boot  7.11M  1.59G  3.83M   784K 20170321 4 min 9 s
             can_you_solve_the_control_room_riddle_dennis_shasha  6.73M  1.59G  4.58M   782K 20160531 4 min 0 s
           can_you_solve_the_dragon_jousting_riddle_alex_gendler  9.13M  1.60G  1.51M   774K 20200109 5 min 3 s
                     why_the_metric_system_matters_matt_anticole  7.17M  1.61G  4.51M   770K 20160721 5 min 7 s
    schrodinger_s_cat_a_thought_experiment_in_quantum_mechanics_  6.93M  1.62G  5.26M   770K 20141014 4 min 37 s
                   can_you_solve_the_dark_coin_riddle_lisa_winer  9.85M  1.63G  3.00M   769K 20180122 5 min 25 s
            can_you_solve_the_seven_planets_riddle_edwin_f_meyer  9.29M  1.63G  2.98M   764K 20180226 6 min 0 s
                   whats_that_ringing_in_your_ears_marc_fagelson  22.4M  1.66G  1.49M   763K 20200817 5 min 38 s
    what_makes_the_great_wall_of_china_so_extraordinary_megan_ca  6.95M  1.66G  4.45M   760K 20150917 4 min 29 s
                              what_causes_headaches_dan_kwartler  9.18M  1.67G  2.95M   755K 20180412 5 min 31 s
       can_you_solve_the_multiplying_rabbits_riddle_alex_gendler  7.66M  1.68G  2.21M   753K 20190110 4 min 38 s
                     why_is_it_so_hard_to_cure_cancer_kyuson_yun  14.5M  1.69G  2.92M   747K 20171010 5 min 22 s
             can_you_solve_the_secret_werewolf_riddle_dan_finkel  7.43M  1.70G  2.19M   746K 20181108 4 min 28 s
                           why_do_people_join_cults_janja_lalich  12.4M  1.71G  3.64M   745K 20170612 6 min 26 s
      can_you_solve_the_leonardo_da_vinci_riddle_tanya_khovanova  7.97M  1.72G  2.90M   743K 20180823 5 min 7 s
             the_wars_that_inspired_game_of_thrones_alex_gendler  11.0M  1.73G  5.07M   742K 20150511 6 min 0 s
                  can_you_solve_the_jail_break_riddle_dan_finkel  7.25M  1.74G  2.15M   735K 20190226 3 min 24 s
             why_do_we_cry_the_three_types_of_tears_alex_gendler  7.77M  1.75G  5.74M   735K 20140226 3 min 58 s
      the_myth_of_loki_and_the_deadly_mistletoe_iseult_gillespie  22.2M  1.77G   734K   734K 20201201 5 min 28 s
    does_your_vote_count_the_electoral_college_explained_christi  11.1M  1.78G  6.44M   733K 20121101 5 min 21 s
         how_we_conquered_the_deadly_smallpox_virus_simona_zompi  9.12M  1.79G  5.67M   726K 20131028 4 min 33 s
      would_you_sacrifice_one_person_to_save_five_eleanor_nelsen  7.06M  1.79G  3.53M   723K 20170112 4 min 55 s
                         what_causes_kidney_stones_arash_shadman  6.68M  1.80G  3.52M   720K 20170703 5 min 14 s
    how_the_world_s_longest_underwater_tunnel_was_built_alex_gen  9.61M  1.81G  1.41M   720K 20200330 5 min 37 s
    the_japanese_folktale_of_the_selfish_scholar_iseult_gillespi  14.7M  1.83G   718K   718K 20200914 4 min 59 s
    why_is_vermeer_s_girl_with_the_pearl_earring_considered_a_ma  8.37M  1.83G  3.50M   718K 20161018 4 min 33 s
                           how_do_solar_panels_work_richard_komp  7.14M  1.84G  4.18M   714K 20160105 4 min 58 s
      why_the_octopus_brain_is_so_extraordinary_claudio_l_guerra  7.17M  1.85G  4.17M   712K 20151203 4 min 16 s
                              what_causes_cavities_mel_rosenberg  9.96M  1.86G  3.47M   710K 20161017 5 min 0 s
                        what_is_bipolar_disorder_helen_m_farrell  8.08M  1.86G  3.45M   706K 20170209 5 min 57 s
                 can_you_solve_the_sea_monster_riddle_dan_finkel  12.4M  1.88G  1.37M   704K 20200402 5 min 27 s
        the_greatest_mathematician_that_never_lived_pratik_aghor  16.1M  1.89G  1.37M   703K 20200706 5 min 12 s
            can_you_solve_the_false_positive_riddle_alex_gendler  10.4M  1.90G  2.74M   701K 20180508 5 min 19 s
                            history_vs_genghis_khan_alex_gendler  10.4M  1.91G  4.78M   699K 20150702 6 min 6 s
    the_dark_ages_how_dark_were_they_really_crash_course_world_h  29.8M  1.94G  6.79M   695K 20120426 12 min 7 s
                         where_did_russia_come_from_alex_gendler  13.9M  1.96G  4.06M   694K 20151013 5 min 19 s
    mary_s_room_a_philosophical_thought_experiment_eleanor_nelse  6.90M  1.96G  3.38M   693K 20170124 4 min 51 s
                ancient_romes_most_notorious_doctor_ramon_glazov  8.17M  1.97G  2.02M   691K 20190718 5 min 10 s
                  how_stress_affects_your_brain_madhumita_murgia  7.50M  1.98G  4.00M   683K 20151109 4 min 15 s
                     how_to_build_a_fictional_world_kate_messner  10.1M  1.99G  5.33M   683K 20140109 5 min 24 s
            can_you_solve_the_riddle_and_escape_hades_dan_finkel  15.3M  2.00G   682K   682K 20201013 4 min 59 s
              no_one_can_figure_out_how_eels_have_sex_lucy_cooke  22.0M  2.02G  1.33M   679K 20200727 6 min 1 s
                can_you_solve_the_cheating_royal_riddle_dan_katz  19.3M  2.04G  1.32M   678K 20200811 5 min 48 s
                 can_you_solve_the_time_travel_riddle_dan_finkel  8.14M  2.05G  1.98M   676K 20181204 4 min 31 s
        the_history_of_the_cuban_missile_crisis_matthew_a_jordan  9.69M  2.06G  3.29M   674K 20160926 4 min 51 s
                    can_you_solve_the_rogue_ai_riddle_dan_finkel  9.02M  2.07G  2.63M   672K 20180807 5 min 12 s
              can_you_outsmart_this_logical_fallacy_alex_gendler  6.02M  2.07G  1.31M   671K 20191125 3 min 44 s
    the_egyptian_myth_of_isis_and_the_seven_scorpions_alex_gendl  8.66M  2.08G  1.31M   671K 20200302 3 min 34 s
                    the_philosophy_of_cynicism_william_d_desmond  10.2M  2.09G  1.31M   669K 20191219 5 min 25 s
      why_should_you_read_tolstoy_s_war_and_peace_brendan_pelsue  10.4M  2.10G  3.26M   669K 20170427 5 min 9 s
                cell_vs_virus_a_battle_for_health_shannon_stiles  10.1M  2.11G  5.20M   666K 20140417 3 min 58 s
                   why_isnt_the_netherlands_underwater_stefan_al  10.3M  2.12G  1.30M   664K 20200324 5 min 23 s
                         how_do_pregnancy_tests_work_tien_nguyen  8.90M  2.13G  4.52M   662K 20150707 4 min 33 s
            how_stress_affects_your_body_sharon_horesh_bergquist  8.32M  2.14G  3.87M   660K 20151022 4 min 42 s
    what_is_imposter_syndrome_and_how_can_you_combat_it_elizabet  6.54M  2.15G  2.57M   657K 20180828 4 min 18 s
         a_day_in_the_life_of_an_ancient_athenian_robert_garland  10.1M  2.16G  2.55M   653K 20180315 5 min 1 s
            the_myth_of_loki_and_the_master_builder_alex_gendler  7.81M  2.16G  1.27M   650K 20191114 4 min 37 s
    what_causes_panic_attacks_and_how_can_you_prevent_them_cindy  17.4M  2.18G   648K   648K 20201008 5 min 22 s
           the_egyptian_myth_of_the_death_of_osiris_alex_gendler  20.5M  2.20G  1.27M   648K 20200716 4 min 14 s
                        platos_allegory_of_the_cave_alex_gendler  13.0M  2.21G  4.42M   646K 20150317 4 min 32 s
                           platos_best_and_worst_ideas_wisecrack  9.76M  2.22G  3.15M   644K 20161025 4 min 48 s
    why_should_you_read_lord_of_the_flies_by_william_golding_jil  12.0M  2.23G  1.26M   643K 20191212 4 min 47 s
                         is_it_bad_to_hold_your_pee_heba_shaheed  6.40M  2.24G  3.14M   643K 20161010 3 min 58 s
              can_you_solve_the_trolls_paradox_riddle_dan_finkel  10.2M  2.25G  1.87M   639K 20181220 3 min 44 s
            why_do_we_love_a_philosophical_inquiry_skye_c_cleary  9.31M  2.26G  3.74M   638K 20160211 5 min 44 s
              can_you_solve_the_vampire_hunter_riddle_dan_finkel  5.88M  2.27G  1.87M   638K 20190128 3 min 51 s
                          a_brief_history_of_dogs_david_ian_howe  9.68M  2.28G  1.86M   635K 20190326 4 min 58 s
       can_you_solve_the_alice_in_wonderland_riddle_alex_gendler  16.9M  2.29G   633K   633K 20201117 5 min 4 s
    why_should_you_listen_to_vivaldi_s_four_seasons_betsy_schwar  8.74M  2.30G  3.09M   632K 20161024 4 min 19 s
             why_should_you_read_fahrenheit_451_iseult_gillespie  9.36M  2.31G  1.84M   628K 20190122 4 min 35 s
                           the_science_of_attraction_dawn_maslar  6.78M  2.32G  4.89M   625K 20140508 4 min 33 s
                   8_traits_of_successful_people_richard_st_john  19.1M  2.33G  5.49M   625K 20130405 7 min 17 s
    can_you_solve_the_cuddly_duddly_fuddly_wuddly_riddle_dan_fin  6.06M  2.34G  1.83M   623K 20190425 3 min 34 s
                      history_vs_napoleon_bonaparte_alex_gendler  8.85M  2.35G  3.65M   622K 20160204 5 min 21 s
    what_s_the_difference_between_accuracy_and_precision_matt_an  7.40M  2.36G  4.24M   620K 20150414 4 min 52 s
                    where_do_superstitions_come_from_stuart_vyse  8.38M  2.36G  3.00M   614K 20170309 5 min 10 s
                          how_a_wound_heals_itself_sarthak_sinha  7.79M  2.37G  4.18M   612K 20141110 4 min 0 s
                                       just_how_small_is_an_atom  14.7M  2.39G  5.96M   611K 20120416 5 min 27 s
                         why_elephants_never_forget_alex_gendler  8.72M  2.40G  4.16M   609K 20141113 5 min 22 s
                     how_does_your_immune_system_work_emma_bryce  9.65M  2.40G  2.38M   608K 20180108 5 min 22 s
                can_you_solve_the_giant_iron_riddle_alex_gendler  5.39M  2.41G  1.78M   607K 20181127 3 min 37 s
          how_do_carbohydrates_impact_your_health_richard_j_wood  8.69M  2.42G  3.55M   606K 20160111 5 min 10 s
                 the_greatest_ted_talk_ever_sold_morgan_spurlock  38.9M  2.46G  5.32M   605K 20130315 19 min 28 s
                    how_does_chemotherapy_work_hyunsoo_joshua_no  12.9M  2.47G  1.18M   603K 20191205 5 min 25 s
           the_treadmill_s_dark_and_twisted_past_conor_heffernan  14.6M  2.48G  3.52M   601K 20150922 4 min 9 s
            how_the_food_you_eat_affects_your_gut_shilpa_ravella  12.3M  2.50G  2.91M   596K 20170323 5 min 9 s
                        how_to_recognize_a_dystopia_alex_gendler  20.0M  2.51G  2.90M   595K 20161115 5 min 55 s
    zen_koans_unsolvable_enigmas_designed_to_break_your_brain_pu  14.2M  2.53G  2.32M   594K 20180816 4 min 57 s
                     why_is_cotton_in_everything_michael_r_stiff  10.4M  2.54G  1.16M   592K 20200123 4 min 53 s
                    what_makes_tattoos_permanent_claudia_aguirre  8.82M  2.55G  4.62M   591K 20140710 4 min 25 s
       how_much_of_what_you_see_is_a_hallucination_elizabeth_cox  12.7M  2.56G  2.31M   591K 20180626 5 min 49 s
            can_you_solve_the_rebel_supplies_riddle_alex_gendler  8.80M  2.57G  1.73M   590K 20180910 5 min 17 s
    what_s_the_fastest_way_to_alphabetize_your_bookshelf_chand_j  6.22M  2.57G  2.88M   589K 20161128 4 min 38 s
                       the_dangers_of_mixing_drugs_celine_valery  14.3M  2.59G  1.15M   589K 20191112 5 min 3 s
     can_you_solve_the_worlds_most_evil_wizard_riddle_dan_finkel  8.54M  2.60G  1.15M   587K 20200519 5 min 16 s
                     how_does_caffeine_keep_us_awake_hanan_qasim  8.05M  2.60G  2.87M   587K 20170717 5 min 14 s
    the_myth_of_the_sampo_an_infinite_source_of_fortune_and_gree  11.9M  2.62G  1.15M   587K 20190923 5 min 15 s
                     how_do_fish_make_electricity_eleanor_nelsen  10.4M  2.63G  2.29M   585K 20171130 5 min 14 s
    everything_changed_when_the_fire_crystal_got_stolen_alex_gen  9.03M  2.64G  1.14M   585K 20200206 4 min 21 s
             will_there_ever_be_a_mile_high_skyscraper_stefan_al  9.17M  2.64G  1.71M   585K 20190207 5 min 4 s
                    the_life_cycle_of_a_cup_of_coffee_a_j_jacobs  20.9M  2.66G   584K   584K 20210104 5 min 4 s
                    debunking_the_myths_of_ocd_natascha_m_santos  9.04M  2.67G  3.98M   583K 20150519 4 min 50 s
                      can_we_create_the_perfect_farm_brent_loken  31.1M  2.70G   581K   581K 20201012 7 min 9 s
              the_rise_and_fall_of_the_inca_empire_gordon_mcewan  14.5M  2.72G  2.27M   581K 20180212 5 min 45 s
              can_you_solve_the_giant_cat_army_riddle_dan_finkel  9.01M  2.73G  2.26M   579K 20180611 5 min 19 s
                             what_is_a_coronavirus_elizabeth_cox  8.85M  2.74G  1.13M   579K 20200514 5 min 15 s
            can_you_solve_the_killer_robo_ants_riddle_dan_finkel  9.24M  2.74G  1.69M   578K 20181009 4 min 51 s
                    how_do_animals_experience_pain_robyn_j_crook  7.80M  2.75G  2.81M   577K 20170117 5 min 6 s
               should_you_trust_unanimous_decisions_derek_abbott  6.56M  2.76G  3.37M   575K 20160418 4 min 2 s
                    why_do_humans_have_a_third_eyelid_dorsa_amir  8.01M  2.77G  1.12M   574K 20191111 4 min 35 s
                the_dark_history_of_iq_tests_stefan_c_dombrowski  12.8M  2.78G  1.12M   573K 20200427 6 min 9 s
    why_should_you_read_one_hundred_years_of_solitude_francisco_  14.1M  2.79G  2.24M   572K 20180830 5 min 30 s
                        why_are_there_so_many_insects_murry_gans  7.45M  2.80G  3.35M   571K 20160301 4 min 43 s
             the_immortal_cells_of_henrietta_lacks_robin_bulleri  9.48M  2.81G  3.35M   571K 20160208 4 min 26 s
            what_yoga_does_to_your_body_and_brain_krishna_sudhir  19.4M  2.83G  1.11M   570K 20200618 6 min 1 s
                                   what_is_entropy_jeff_phillips  7.75M  2.84G  2.78M   569K 20170509 5 min 19 s
    how_do_steroids_affect_your_muscles_and_the_rest_of_your_bod  18.1M  2.85G   569K   569K 20201109 5 min 48 s
                 are_all_of_your_memories_real_daniel_l_schacter  18.5M  2.87G   563K   563K 20200908 5 min 17 s
         can_you_solve_the_buried_treasure_riddle_daniel_griller  9.47M  2.88G  2.20M   562K 20180322 5 min 2 s
              can_you_solve_the_monster_duel_riddle_alex_gendler  16.3M  2.90G   560K   560K 20201208 5 min 35 s
               the_chinese_myth_of_the_meddling_monk_shunan_teng  8.00M  2.90G  1.63M   558K 20190523 4 min 3 s
                               what_causes_insomnia_dan_kwartler  7.39M  2.91G  2.17M   557K 20180614 5 min 11 s
      the_history_of_the_world_according_to_corn_chris_a_kniesly  14.0M  2.93G  1.08M   554K 20191126 5 min 22 s
                      why_should_you_read_macbeth_brendan_pelsue  10.5M  2.94G  2.13M   545K 20171102 6 min 8 s
       is_there_any_truth_to_the_king_arthur_legends_alan_lupack  15.0M  2.95G  1.59M   542K 20180911 5 min 42 s
                         why_do_your_knuckles_pop_eleanor_nelsen  12.1M  2.96G  3.69M   540K 20150505 4 min 21 s
    how_high_can_you_count_on_your_fingers_spoiler_much_higher_t  9.61M  2.97G  2.63M   539K 20161215 4 min 29 s
           why_should_you_read_crime_and_punishment_alex_gendler  8.22M  2.98G  1.57M   538K 20190514 4 min 45 s
    how_did_dracula_become_the_world_s_most_famous_vampire_stanl  10.5M  2.99G  2.62M   536K 20170420 5 min 5 s
              the_most_successful_pirate_of_all_time_dian_murray  8.43M  3.00G  2.08M   532K 20180402 5 min 16 s
    the_journey_to_pluto_the_farthest_world_ever_explored_alan_s  8.63M  3.01G  2.07M   530K 20180517 6 min 9 s
         what_is_the_heisenberg_uncertainty_principle_chad_orzel  5.96M  3.01G  3.62M   530K 20140916 4 min 43 s
                       at_what_moment_are_you_dead_randall_hayes  9.25M  3.02G  3.61M   528K 20141211 5 min 33 s
                        the_prison_break_think_like_a_coder_ep_1  14.0M  3.03G  1.03M   526K 20190930 6 min 50 s
                        how_blood_pressure_works_wilfred_manzano  8.16M  3.04G  3.59M   525K 20150723 4 min 31 s
                       where_did_english_come_from_claire_bowern  9.49M  3.05G  3.58M   524K 20150716 4 min 53 s
                 the_benefits_of_a_good_night_s_sleep_shai_marcu  11.3M  3.06G  3.58M   524K 20150105 5 min 44 s
              a_day_in_the_life_of_a_celtic_druid_philip_freeman  15.0M  3.08G  1.02M   522K 20190917 4 min 30 s
                        how_do_vitamins_work_ginnie_trinh_nguyen  9.07M  3.09G  3.56M   521K 20141006 4 min 43 s
                 can_you_solve_the_alien_probe_riddle_dan_finkel  8.42M  3.09G  1.52M   520K 20180924 5 min 8 s
                                  is_telekinesis_real_emma_bryce  8.68M  3.10G  3.54M   518K 20140930 5 min 22 s
                     did_the_amazons_really_exist_adrienne_mayor  14.0M  3.12G  2.02M   517K 20180618 5 min 5 s
    how_many_ways_are_there_to_prove_the_pythagorean_theorem_bet  7.39M  3.12G  2.01M   514K 20170911 5 min 16 s
                 is_fire_a_solid_a_liquid_or_a_gas_elizabeth_cox  9.93M  3.13G  1.50M   513K 20181105 4 min 34 s
             how_squids_outsmart_their_predators_carly_anne_york  9.54M  3.14G  2.00M   512K 20180514 5 min 12 s
                             what_causes_body_odor_mel_rosenberg  6.32M  3.15G  2.00M   512K 20180405 4 min 28 s
                          the_sonic_boom_problem_katerina_kaouri  11.3M  3.16G  3.48M   509K 20150210 5 min 43 s
                              a_brief_history_of_goths_dan_adams  11.8M  3.17G  2.49M   509K 20170518 5 min 30 s
    why_do_competitors_open_their_stores_next_to_one_another_jac  8.49M  3.18G  4.47M   509K 20121001 4 min 6 s
       the_rise_and_fall_of_the_assyrian_empire_marian_h_feldman  12.1M  3.19G  1.99M   509K 20180424 5 min 15 s
                       how_to_write_descriptively_nalo_hopkinson  12.1M  3.20G  2.97M   507K 20151116 4 min 41 s
                                   how_to_make_a_mummy_len_bloch  7.41M  3.21G  3.43M   502K 20150618 4 min 45 s
     the_amazing_ways_plants_defend_themselves_valentin_hammoudi  11.5M  3.22G  2.45M   502K 20170828 6 min 11 s
               how_rollercoasters_affect_your_body_brian_d_avery  9.21M  3.23G  1.47M   501K 20181029 5 min 1 s
        how_parasites_change_their_host_s_behavior_jaap_de_roode  10.1M  3.24G  3.42M   501K 20150309 5 min 13 s
                    if_superpowers_were_real_immortality_joy_lin  14.3M  3.26G  4.40M   500K 20130627 4 min 29 s
               the_pharaoh_that_wouldn_t_be_forgotten_kate_green  9.09M  3.26G  3.42M   500K 20141215 4 min 33 s
              how_to_outsmart_the_prisoners_dilemma_lucas_husted  22.9M  3.29G   995K   498K 20200827 5 min 44 s
    the_myth_of_oisin_and_the_land_of_eternal_youth_iseult_gille  8.19M  3.29G  1.94M   497K 20180118 3 min 49 s
                  what_caused_the_french_revolution_tom_mullaney  13.9M  3.31G  2.40M   492K 20161027 5 min 38 s
          what_is_the_tragedy_of_the_commons_nicholas_amendolare  12.4M  3.32G  1.92M   491K 20171121 4 min 57 s
                             how_to_unboil_an_egg_eleanor_nelsen  7.29M  3.33G  3.34M   488K 20150423 4 min 9 s
                                    how_to_read_music_tim_hansen  8.54M  3.34G  4.28M   487K 20130718 5 min 23 s
                          history_vs_vladimir_lenin_alex_gendler  15.9M  3.35G  3.80M   487K 20140407 6 min 42 s
                can_you_solve_the_death_race_riddle_alex_gendler  13.6M  3.36G   973K   486K 20200303 5 min 50 s
               the_art_forger_who_tricked_the_nazis_noah_charney  10.4M  3.37G   971K   486K 20200406 5 min 16 s
                                      string_theory_brian_greene  37.8M  3.41G  4.27M   485K 20130809 19 min 9 s
              the_scientific_origins_of_the_minotaur_matt_kaplan  10.4M  3.42G  3.32M   485K 20150720 4 min 40 s
                         the_genius_of_marie_curie_shohini_ghose  9.06M  3.43G  2.37M   484K 20170608 5 min 3 s
                           your_body_vs_implants_kaitlyn_sadtler  11.4M  3.44G  1.41M   482K 20190617 4 min 39 s
               the_myth_of_the_stolen_eyeballs_nathan_d_horowitz  23.6M  3.46G   481K   481K 20201005 5 min 29 s
    what_really_happens_to_the_plastic_you_throw_away_emma_bryce  6.68M  3.47G  3.29M   481K 20150421 4 min 6 s
                    history_vs_christopher_columbus_alex_gendler  10.0M  3.48G  3.28M   480K 20141013 5 min 54 s
     how_the_normans_changed_the_history_of_europe_mark_robinson  13.3M  3.49G  1.87M   480K 20180809 5 min 19 s
    exponential_growth_how_folding_paper_can_get_you_to_the_moon  6.30M  3.50G  4.68M   479K 20120419 3 min 48 s
    are_you_a_body_with_a_mind_or_a_mind_with_a_body_maryam_alim  10.6M  3.51G  1.87M   479K 20170925 6 min 9 s
         the_real_story_behind_archimedes_eureka_armand_d_angour  10.3M  3.52G  3.26M   477K 20150313 4 min 41 s
       a_day_in_the_life_of_a_mongolian_queen_anne_f_broadbridge  7.17M  3.53G  1.39M   476K 20190117 4 min 23 s
               how_does_your_body_process_medicine_celine_valery  9.04M  3.54G  2.32M   475K 20170515 4 min 12 s
    why_dont_poisonous_animals_poison_themselves_rebecca_d_tarvi  9.25M  3.55G  1.85M   474K 20180705 5 min 9 s
                          history_vs_sigmund_freud_todd_dufresne  8.79M  3.55G   947K   473K 20200331 5 min 54 s
            why_do_we_harvest_horseshoe_crab_blood_elizabeth_cox  7.50M  3.56G  1.85M   473K 20170921 4 min 45 s
          how_the_monkey_king_escaped_the_underworld_shunan_teng  15.4M  3.58G   946K   473K 20200407 4 min 57 s
                      the_power_of_the_placebo_effect_emma_bryce  8.45M  3.58G  2.76M   471K 20160404 4 min 37 s
    what_causes_opioid_addiction_and_why_is_it_so_tough_to_comba  19.1M  3.60G   939K   470K 20200507 8 min 21 s
    this_is_sparta_fierce_warriors_of_the_ancient_world_craig_zi  8.43M  3.61G  2.75M   469K 20160308 4 min 27 s
    game_theory_challenge_can_you_predict_human_behavior_lucas_h  5.95M  3.62G   936K   468K 20191105 4 min 58 s
    can_you_solve_the_sorting_hat_riddle_dan_katz_and_alex_rosen  20.2M  3.64G   935K   468K 20200901 5 min 59 s
                       how_aspirin_was_discovered_krishna_sudhir  11.6M  3.65G  1.81M   464K 20171002 5 min 44 s
          why_doesnt_anything_stick_to_teflon_ashwini_bharathula  8.67M  3.66G  2.26M   462K 20161213 4 min 44 s
            the_evolution_of_animal_genitalia_menno_schilthuizen  13.2M  3.67G  2.26M   462K 20170424 5 min 35 s
    the_rise_and_fall_of_historys_first_empire_soraya_field_fior  23.9M  3.69G   461K   461K 20201015 5 min 37 s
               a_brief_history_of_banned_numbers_alessandra_king  7.66M  3.70G  1.79M   459K 20170926 4 min 42 s
                   why_is_meningitis_so_dangerous_melvin_sanicas  9.77M  3.71G  1.34M   458K 20181119 4 min 53 s
                     how_fast_can_a_vaccine_be_made_dan_kwartler  20.4M  3.73G   916K   458K 20200615 5 min 51 s
    the_incredible_history_of_china_s_terracotta_warriors_megan_  7.41M  3.74G  3.13M   458K 20150630 4 min 31 s
                                      a_brief_history_of_plastic  17.1M  3.75G   457K   457K 20200910 5 min 40 s
                   how_fast_is_the_speed_of_thought_seena_mathew  20.8M  3.77G   457K   457K 20201116 5 min 16 s
                         ugly_history_witch_hunts_brian_a_pavlac  13.9M  3.79G  1.33M   455K 20190611 5 min 25 s
                           what_is_dyslexia_kelli_sandman_hurley  7.51M  3.79G  3.98M   452K 20130715 4 min 34 s
             why_should_you_read_james_joyce_s_ulysses_sam_slote  18.6M  3.81G  1.77M   452K 20171024 5 min 58 s
                        how_do_personality_tests_work_merve_emre  19.5M  3.83G   451K   451K 20201222 4 min 56 s
    dna_hot_pockets_the_longest_word_ever_crash_course_biology_1  44.7M  3.88G  4.39M   450K 20120409 14 min 7 s
    titan_of_terror_the_dark_imagination_of_h_p_lovecraft_silvia  8.52M  3.88G  1.32M   449K 20190423 4 min 46 s
     can_you_outsmart_a_troll_by_thinking_like_one_claire_wardle  18.4M  3.90G   445K   445K 20201029 5 min 0 s
    the_surprising_reason_you_feel_awful_when_you_re_sick_marco_  7.36M  3.91G  2.60M   444K 20160419 5 min 0 s
    what_machiavellian_really_means_pazit_cahlon_and_alex_gendle  7.85M  3.92G  1.30M   442K 20190325 5 min 9 s
           the_surprising_reasons_animals_play_dead_tierney_thys  7.38M  3.92G  1.71M   438K 20180416 4 min 46 s
       the_aztec_myth_of_the_unlikeliest_sun_god_kay_almere_read  12.1M  3.94G  1.28M   437K 20190509 4 min 12 s
    jellyfish_predate_dinosaurs_how_have_they_survived_so_long_d  11.0M  3.95G  2.13M   436K 20170328 5 min 25 s
    ethical_dilemma_the_burger_murders_george_siedel_and_christi  21.9M  3.97G   871K   435K 20200728 5 min 45 s
     why_is_herodotus_called_the_father_of_history_mark_robinson  7.87M  3.98G  1.68M   431K 20171211 5 min 2 s
          how_do_viruses_jump_from_animals_to_humans_ben_longdon  10.4M  3.99G  1.26M   430K 20190808 5 min 4 s
                         why_is_glass_transparent_mark_miodownik  6.71M  3.99G  3.35M   429K 20140204 4 min 7 s
            history_vs_henry_viii_mark_robinson_and_alex_gendler  7.82M  4.00G  1.25M   428K 20181112 5 min 24 s
    would_you_opt_for_a_life_with_no_pain_hayley_levitt_and_beth  6.52M  4.01G  2.50M   427K 20151117 4 min 9 s
       what_is_mccarthyism_and_how_did_it_happen_ellen_schrecker  9.78M  4.02G  2.08M   427K 20170314 5 min 42 s
     how_to_use_rhetoric_to_get_what_you_want_camille_a_langston  6.38M  4.02G  2.08M   425K 20160920 4 min 29 s
     exploring_other_dimensions_alex_rosenthal_and_george_zaidan  8.57M  4.03G  3.70M   420K 20130717 4 min 48 s
                           are_ghost_ships_real_peter_b_campbell  11.7M  4.04G  2.05M   419K 20170228 5 min 0 s
    which_bag_should_you_use_luka_seamus_wright_and_imogen_ellen  14.6M  4.06G   419K   419K 20201119 4 min 52 s
     how_do_nuclear_power_plants_work_m_v_ramana_and_sajan_saini  11.5M  4.07G  2.03M   415K 20170508 8 min 6 s
                           history_vs_richard_nixon_alex_gendler  9.98M  4.08G  2.83M   414K 20150212 5 min 39 s
                    what_happens_during_a_stroke_vaibhav_goswami  10.5M  4.09G  1.62M   414K 20180201 4 min 59 s
           what_happens_when_you_remove_the_hippocampus_sam_kean  7.69M  4.10G  3.21M   411K 20140826 5 min 25 s
                            is_time_travel_possible_colin_stuart  11.5M  4.11G  3.21M   411K 20131021 5 min 3 s
             why_do_airlines_sell_too_many_tickets_nina_klietsch  6.11M  4.11G  2.00M   410K 20161220 4 min 59 s
           jabberwocky_one_of_literature_s_best_bits_of_nonsense  10.7M  4.12G   409K   409K 20200917 2 min 10 s
                     can_a_black_hole_be_destroyed_fabio_pacucci  16.3M  4.14G  1.19M   407K 20190516 5 min 6 s
    what_really_happened_to_the_library_of_alexandria_elizabeth_  8.42M  4.15G  1.59M   407K 20180814 4 min 52 s
                            the_history_of_marriage_alex_gendler  8.24M  4.15G  3.17M   405K 20140213 4 min 43 s
    a_day_in_the_life_of_a_teenage_samurai_constantine_n_vaporis  36.1M  4.19G   810K   405K 20200622 5 min 30 s
                    human_sperm_vs_the_sperm_whale_aatish_bhatia  10.1M  4.20G  3.16M   404K 20130923 4 min 17 s
             the_wacky_history_of_cell_theory_lauren_royal_woods  15.4M  4.22G  3.91M   401K 20120604 6 min 11 s
                    will_we_ever_be_able_to_teleport_sajan_saini  17.3M  4.23G  1.95M   400K 20170731 5 min 37 s
             how_to_make_your_writing_suspenseful_victoria_smith  8.44M  4.24G  1.56M   398K 20171031 4 min 35 s
                  the_hidden_treasures_of_timbuktu_elizabeth_cox  20.6M  4.26G   398K   398K 20201022 5 min 34 s
                           the_science_of_spiciness_rose_eveleth  6.03M  4.27G  3.09M   396K 20140310 3 min 54 s
                the_taino_myth_of_the_cursed_creator_bill_keegan  10.9M  4.28G   786K   393K 20191104 3 min 45 s
      what_percentage_of_your_brain_do_you_use_richard_e_cytowic  8.93M  4.29G  3.06M   391K 20140130 5 min 15 s
    can_you_survive_nuclear_fallout_brooke_buddemeier_and_jessic  8.90M  4.29G  1.15M   391K 20190108 5 min 24 s
              the_hidden_meanings_of_yin_and_yang_john_bellaimey  13.5M  4.31G  3.44M   391K 20130802 4 min 9 s
              where_do_math_symbols_come_from_john_david_walters  10.6M  4.32G  1.52M   389K 20171030 4 min 29 s
            football_physics_the_impossible_free_kick_erez_garty  7.58M  4.33G  2.66M   389K 20150615 3 min 32 s
                    can_you_solve_the_honeybee_riddle_dan_finkel  17.3M  4.34G   776K   388K 20200730 5 min 32 s
    can_100_renewable_energy_power_the_world_federico_rosei_and_  9.36M  4.35G  1.50M   384K 20171207 5 min 54 s
         check_your_intuition_the_birthday_problem_david_knuffke  13.3M  4.36G  1.87M   384K 20170504 5 min 6 s
            the_myth_of_jason_and_the_argonauts_iseult_gillespie  13.5M  4.38G  1.12M   383K 20190827 5 min 14 s
                         the_magic_of_vedic_math_gaurav_tekriwal  19.2M  4.40G  3.36M   382K 20130416 9 min 44 s
                    the_science_of_skin_color_angela_koine_flynn  9.01M  4.41G  2.24M   382K 20160216 4 min 53 s
         the_physics_of_the_hardest_move_in_ballet_arleen_sugano  7.90M  4.41G  2.23M   381K 20160322 4 min 16 s
    what_happens_if_you_cut_down_all_of_a_city_s_trees_stefan_al  14.9M  4.43G   760K   380K 20200424 5 min 25 s
                       how_computer_memory_works_kanawat_senanan  13.2M  4.44G  2.21M   378K 20160510 5 min 4 s
                           what_causes_constipation_heba_shaheed  5.30M  4.45G  1.47M   376K 20180507 3 min 32 s
                    three_ways_the_universe_could_end_venus_keus  7.53M  4.45G  1.10M   375K 20190219 4 min 46 s
                         the_life_cycle_of_a_t_shirt_angel_chang  12.3M  4.46G  1.83M   374K 20170905 6 min 3 s
    how_a_single_celled_organism_almost_wiped_out_life_on_earth_  7.87M  4.47G  2.19M   374K 20160811 4 min 13 s
                    how_to_spot_a_misleading_graph_lea_gaslowitz  8.34M  4.48G  1.82M   373K 20170706 4 min 9 s
                the_princess_who_rewrote_history_leonora_neville  7.54M  4.49G  1.09M   373K 20181022 4 min 57 s
          why_should_you_read_the_handmaid_s_tale_naomi_r_mercer  11.2M  4.50G  1.46M   373K 20180308 5 min 4 s
    how_one_scientist_averted_a_national_health_crisis_andrea_to  13.0M  4.51G  1.46M   373K 20180607 5 min 31 s
             why_should_you_read_virginia_woolf_iseult_gillespie  15.7M  4.53G  1.45M   371K 20171005 6 min 2 s
    a_day_in_the_life_of_an_ancient_babylonian_business_mogul_so  22.4M  4.55G   369K   369K 20210107 4 min 46 s
                   how_do_investors_choose_stocks_richard_coffin  15.1M  4.56G   369K   369K 20201110 5 min 1 s
                           why_can_t_some_birds_fly_gillian_gibb  6.71M  4.57G  1.08M   368K 20181018 4 min 31 s
                            how_do_ventilators_work_alex_gendler  12.1M  4.58G   735K   367K 20200521 5 min 41 s
               the_wild_world_of_carnivorous_plants_kenny_coogan  7.90M  4.59G  1.07M   367K 20190411 5 min 10 s
                     why_is_mount_everest_so_tall_michele_koppes  7.35M  4.60G  2.14M   365K 20160407 4 min 52 s
                              how_smart_are_dolphins_lori_marino  9.95M  4.61G  2.49M   365K 20150831 4 min 50 s
               how_does_your_body_know_you_re_full_hilary_coller  8.23M  4.61G  1.42M   364K 20171113 4 min 33 s
               vampires_folklore_fantasy_and_fact_michael_molina  11.5M  4.63G  2.84M   364K 20131029 6 min 56 s
                               how_do_crystals_work_graham_baird  12.4M  4.64G  1.07M   364K 20190618 5 min 6 s
                      the_worlds_largest_organism_alex_rosenthal  23.3M  4.66G   364K   364K 20201207 6 min 27 s
             sex_determination_more_complicated_than_you_thought  17.4M  4.68G  3.55M   363K 20120423 5 min 45 s
               why_should_you_read_edgar_allan_poe_scott_peeples  12.5M  4.69G  1.06M   363K 20180918 4 min 55 s
          why_do_you_get_a_fever_when_you_re_sick_christian_moro  19.7M  4.71G   363K   363K 20201112 5 min 37 s
    the_imaginary_king_who_changed_the_real_world_matteo_salvado  12.1M  4.72G   718K   359K 20200319 5 min 33 s
                               the_maya_myth_of_the_morning_star  8.79M  4.73G   715K   357K 20191021 4 min 35 s
    why_should_you_read_dantes_divine_comedy_sheila_marie_orfano  12.2M  4.74G   715K   357K 20191010 5 min 9 s
                  the_wicked_wit_of_jane_austen_iseult_gillespie  15.8M  4.76G  1.05M   357K 20190321 5 min 0 s
              poison_vs_venom_what_s_the_difference_rose_eveleth  7.73M  4.76G  2.78M   356K 20140220 3 min 55 s
                                   why_do_we_hiccup_john_cameron  10.1M  4.77G  2.08M   354K 20160728 4 min 49 s
          why_are_we_so_attached_to_our_things_christian_jarrett  8.22M  4.78G  1.73M   353K 20161227 4 min 34 s
                   how_the_heart_actually_pumps_blood_edmond_hui  6.19M  4.79G  2.76M   353K 20140520 4 min 27 s
                  what_is_deja_vu_what_is_deja_vu_michael_molina  5.50M  4.79G  3.10M   353K 20130828 3 min 54 s
                         what_orwellian_really_means_noah_tavlin  18.6M  4.81G  2.06M   352K 20151001 5 min 31 s
      what_happened_when_we_all_stopped_narrated_by_jane_goodall  6.08M  4.82G   703K   352K 20200604 3 min 15 s
               the_surprising_cause_of_stomach_ulcers_rusha_modi  11.6M  4.83G  1.37M   351K 20170928 5 min 34 s
               the_strange_case_of_the_cyclops_sheep_tien_nguyen  6.87M  4.84G  1.37M   351K 20171003 4 min 40 s
                                   claws_vs_nails_matthew_borths  7.38M  4.84G   701K   351K 20191029 5 min 10 s
    building_the_world_s_largest_and_most_controversial_power_pl  22.3M  4.86G   350K   350K 20201210 5 min 37 s
                                 how_smart_are_orangutans_lu_gao  6.85M  4.87G  2.05M   350K 20160830 4 min 32 s
    the_mathematical_secrets_of_pascals_triangle_wajdi_mohamed_r  6.80M  4.88G  2.04M   348K 20150915 4 min 49 s
                       why_do_blood_types_matter_natalie_s_hodge  9.45M  4.89G  2.38M   348K 20150629 4 min 41 s
                      the_arctic_vs_the_antarctic_camille_seaman  6.20M  4.89G  3.05M   347K 20130819 4 min 24 s
    how_does_the_nobel_peace_prize_work_adeline_cuvelier_and_tor  11.7M  4.90G  1.69M   347K 20161006 6 min 14 s
                  a_day_in_the_life_of_an_aztec_midwife_kay_read  7.59M  4.91G   693K   347K 20200512 4 min 35 s
                  what_is_zeno_s_dichotomy_paradox_colm_kelleher  8.55M  4.92G  3.05M   347K 20130415 4 min 11 s
                    the_mysterious_science_of_pain_joshua_w_pate  10.8M  4.93G  1.01M   346K 20190520 5 min 2 s
    can_you_outsmart_the_fallacy_that_fooled_a_generation_of_doc  19.7M  4.95G   691K   346K 20200810 5 min 30 s
    how_can_you_change_someone_s_mind_hint_facts_aren_t_always_e  10.5M  4.96G  1.35M   345K 20180726 4 min 39 s
                 inside_the_killer_whale_matriarchy_darren_croft  13.1M  4.97G  1.01M   344K 20181211 5 min 3 s
             what_does_this_symbol_actually_mean_adrian_treharne  8.34M  4.98G  1.67M   342K 20170105 4 min 10 s
    the_meaning_of_life_according_to_simone_de_beauvoir_iseult_g  12.3M  4.99G   682K   341K 20200310 5 min 10 s
                                    what_is_a_calorie_emma_bryce  5.65M  5.00G  2.33M   340K 20150713 4 min 11 s
    the_murder_of_ancient_alexandria_s_greatest_scholar_soraya_f  13.5M  5.01G  1.00M   340K 20190801 5 min 4 s
         the_science_behind_the_myth_homer_s_odyssey_matt_kaplan  6.50M  5.02G  1.99M   340K 20151110 4 min 31 s
    what_makes_tuberculosis_tb_the_world_s_most_infectious_kille  7.24M  5.03G  0.99M   340K 20190627 5 min 17 s
                     who_am_i_a_philosophical_inquiry_amy_adkins  10.6M  5.04G  2.32M   339K 20150811 4 min 58 s
                            the_paradox_of_value_akshita_agarwal  7.73M  5.04G  1.98M   338K 20160829 3 min 45 s
                          how_does_impeachment_work_alex_gendler  11.3M  5.05G  1.65M   337K 20170824 5 min 12 s
    are_elvish_klingon_dothraki_and_na_vi_real_languages_john_mc  8.70M  5.06G  2.62M   336K 20130926 5 min 20 s
                    are_we_living_in_a_simulation_zohreh_davoudi  5.98M  5.07G   668K   334K 20191008 4 min 23 s
    vultures_the_acid_puking_plague_busting_heroes_of_the_ecosys  9.34M  5.08G   667K   334K 20200224 5 min 5 s
    the_ferocious_predatory_dinosaurs_of_cretaceous_sahara_nizar  8.82M  5.09G  1.62M   333K 20170606 4 min 19 s
                    why_should_you_read_don_quixote_ilan_stavans  11.9M  5.10G   997K   332K 20181008 5 min 38 s
                        why_do_some_people_go_bald_sarthak_sinha  7.62M  5.11G  2.26M   330K 20150825 4 min 48 s
                      what_causes_antibiotic_resistance_kevin_wu  8.13M  5.11G  2.57M   329K 20140807 4 min 34 s
                    why_is_ketchup_so_hard_to_pour_george_zaidan  10.0M  5.12G  2.57M   328K 20140408 4 min 28 s
                what_causes_an_economic_recession_richard_coffin  7.43M  5.13G   657K   328K 20191015 5 min 4 s
    the_most_groundbreaking_scientist_you_ve_never_heard_of_addi  6.87M  5.14G  2.56M   328K 20131001 4 min 32 s
                       how_your_muscular_system_works_emma_bryce  8.63M  5.15G  1.28M   328K 20171026 4 min 44 s
                   how_to_defeat_a_dragon_with_math_garth_sundem  8.27M  5.15G  2.86M   326K 20130115 3 min 46 s
                                how_to_understand_power_eric_liu  14.7M  5.17G  2.22M   325K 20141104 7 min 1 s
                         what_makes_a_poem_a_poem_melissa_kovacs  15.5M  5.18G  1.59M   325K 20170320 5 min 19 s
                what_gives_a_dollar_bill_its_value_doug_levinson  7.24M  5.19G  2.53M   324K 20140623 3 min 51 s
                       the_true_story_of_sacajawea_karen_mensing  5.88M  5.20G  2.85M   324K 20130808 3 min 40 s
                                   should_we_eat_bugs_emma_bryce  10.2M  5.21G  2.52M   323K 20140102 4 min 51 s
                       the_chemistry_of_cookies_stephanie_warren  6.33M  5.21G  2.52M   322K 20131119 4 min 29 s
                history_vs_augustus_peta_greenfield_alex_gendler  8.30M  5.22G  1.26M   322K 20180717 5 min 10 s
       ugly_history_japanese_american_incarceration_camps_densho  9.86M  5.23G   643K   322K 20191001 5 min 45 s
    the_strange_history_of_the_world_s_most_stolen_painting_noah  23.4M  5.25G   322K   322K 20201221 5 min 37 s
    turbulence_one_of_the_great_unsolved_mysteries_of_physics_to  13.1M  5.27G   962K   321K 20190415 5 min 27 s
    can_you_solve_the_multiverse_rescue_mission_riddle_dan_finke  13.3M  5.28G   959K   320K 20190815 4 min 24 s
     what_if_cracks_in_concrete_could_fix_themselves_congrui_jin  8.21M  5.29G   958K   319K 20181016 4 min 39 s
    light_seconds_light_years_light_centuries_how_to_measure_ext  16.7M  5.30G  2.18M   319K 20141009 5 min 29 s
    can_you_outsmart_the_fallacy_that_started_a_witch_hunt_eliza  16.8M  5.32G   317K   317K 20201026 4 min 23 s
                     how_do_ocean_currents_work_jennifer_verduin  10.2M  5.33G   943K   314K 20190131 4 min 33 s
         the_irish_myth_of_the_giant_s_causeway_iseult_gillespie  7.27M  5.34G  1.23M   314K 20180612 3 min 48 s
                 what_s_invisible_more_than_you_think_john_lloyd  19.8M  5.36G  2.76M   314K 20120926 8 min 47 s
                        how_do_pain_relievers_work_george_zaidan  8.16M  5.36G  3.06M   314K 20120626 4 min 13 s
    why_does_your_voice_change_as_you_get_older_shaylin_a_schund  11.9M  5.38G  1.22M   313K 20180802 5 min 5 s
    oxygens_surprisingly_complex_journey_through_your_body_enda_  10.3M  5.39G  1.52M   312K 20170413 5 min 9 s
           making_a_ted_ed_lesson_bringing_a_pop_up_book_to_life  15.1M  5.40G  2.13M   311K 20140910 6 min 12 s
                   the_life_cycle_of_a_neutron_star_david_lunney  11.0M  5.41G   932K   311K 20181120 5 min 16 s
                         how_do_hard_drives_work_kanawat_senanan  17.8M  5.43G  1.82M   310K 20151029 5 min 11 s
                            what_causes_bad_breath_mel_rosenberg  8.75M  5.44G  2.12M   310K 20150331 4 min 13 s
                did_ancient_troy_really_exist_einav_zamir_dembin  9.00M  5.45G  1.21M   308K 20180731 4 min 37 s
    how_do_you_decide_where_to_go_in_a_zombie_apocalypse_david_h  5.98M  5.45G  2.70M   307K 20130603 3 min 37 s
               the_complex_geometry_of_islamic_design_eric_broug  17.1M  5.47G  2.08M   304K 20150514 5 min 6 s
                 what_is_the_universe_expanding_into_sajan_saini  12.3M  5.48G  1.18M   303K 20180906 6 min 6 s
    the_otherworldly_creatures_in_the_ocean_s_deepest_depths_lid  7.37M  5.49G  1.76M   301K 20160524 5 min 2 s
                   whats_the_big_deal_with_gluten_william_d_chey  7.71M  5.50G  2.05M   300K 20150602 5 min 17 s
                            how_does_fracking_work_mia_nacamulli  8.50M  5.50G  1.45M   298K 20170713 6 min 3 s
         why_is_pneumonia_so_dangerous_eve_gaus_and_vanessa_ruiz  15.3M  5.52G   298K   298K 20201130 4 min 27 s
                                            how_pandemics_spread  18.7M  5.54G  2.91M   298K 20120311 7 min 59 s
        frida_kahlo_the_woman_behind_the_legend_iseult_gillespie  11.5M  5.55G   888K   296K 20190328 4 min 6 s
                            how_big_is_infinity_dennis_wildfogel  11.5M  5.56G  2.87M   294K 20120806 7 min 12 s
                       the_science_of_milk_jonathan_j_o_sullivan  9.02M  5.57G  1.43M   294K 20170131 5 min 23 s
        the_secret_student_resistance_to_hitler_iseult_gillespie  9.19M  5.58G   878K   293K 20190903 5 min 32 s
                    which_voting_system_is_the_best_alex_gendler  18.0M  5.59G   585K   293K 20200611 5 min 32 s
                 why_should_you_read_kurt_vonnegut_mia_nacamulli  10.6M  5.60G   876K   292K 20181129 5 min 23 s
          how_memories_form_and_how_we_lose_them_catharine_young  7.48M  5.61G  1.71M   292K 20150924 4 min 19 s
                              the_road_not_taken_by_robert_frost  3.83M  5.62G   874K   291K 20190202 2 min 11 s
                               how_languages_evolve_alex_gendler  7.30M  5.62G  2.27M   291K 20140527 4 min 2 s
             the_problem_with_the_u_s_bail_system_camilo_ramirez  23.8M  5.65G   290K   290K 20200929 6 min 29 s
                   why_is_biodiversity_so_important_kim_preshoff  10.6M  5.66G  1.98M   290K 20150420 4 min 18 s
                               who_is_sherlock_holmes_neil_mccaw  8.34M  5.66G  1.69M   289K 20160505 4 min 53 s
                    why_its_so_hard_to_cure_hiv_aids_janet_iwasa  7.72M  5.67G  1.97M   288K 20150316 4 min 30 s
         music_and_math_the_genius_of_beethoven_natalya_st_clair  10.2M  5.68G  1.96M   287K 20140909 4 min 19 s
                        einstein_s_miracle_year_larry_lagerstrom  7.55M  5.69G  1.96M   287K 20150106 5 min 15 s
                      how_tall_can_a_tree_grow_valentin_hammoudi  8.21M  5.70G   859K   286K 20190314 4 min 45 s
    one_of_the_most_difficult_words_to_translate_krystian_aparta  6.00M  5.70G  1.68M   286K 20160906 3 min 47 s
                                what_causes_heartburn_rusha_modi  11.5M  5.71G   856K   285K 20181101 4 min 54 s
    what_is_a_butt_tuba_and_why_is_it_in_medieval_art_michelle_b  6.83M  5.72G   854K   285K 20190416 4 min 40 s
                       the_deadly_irony_of_gunpowder_eric_rosado  8.75M  5.73G  2.22M   285K 20131104 3 min 24 s
            can_you_solve_the_dark_matter_fuel_riddle_dan_finkel  7.74M  5.74G   853K   284K 20190716 4 min 24 s
    how_one_piece_of_legislation_divided_a_nation_ben_labaree_jr  13.8M  5.75G  2.21M   282K 20140211 6 min 2 s
       what_is_the_coldest_thing_in_the_world_lina_marieth_hoyos  5.77M  5.76G  1.10M   282K 20180710 4 min 25 s
                                             how_far_is_a_second  2.58M  5.76G  2.75M   281K 20120407 1 min 29 s
                   how_do_vaccines_work_kelwalin_dhanasarnsombut  6.41M  5.77G  1.92M   281K 20150112 4 min 35 s
      history_through_the_eyes_of_the_potato_leo_bear_mcguinness  7.60M  5.77G  1.65M   281K 20151214 3 min 46 s
    how_do_glasses_help_us_see_andrew_bastawrous_and_clare_gilbe  11.8M  5.78G  1.65M   281K 20160405 4 min 23 s
                         why_can_t_we_see_evidence_of_alien_life  15.4M  5.80G  2.74M   281K 20120311 6 min 3 s
             when_will_the_next_ice_age_happen_lorraine_lisiecki  8.03M  5.81G  1.09M   280K 20180510 5 min 6 s
    how_mendel_s_pea_plants_helped_us_understand_genetics_horten  5.11M  5.81G  2.45M   278K 20130312 3 min 6 s
                 do_larger_animals_take_longer_to_pee_david_l_hu  14.2M  5.83G   278K   278K 20201105 4 min 46 s
                 how_simple_ideas_lead_to_scientific_discoveries  14.9M  5.84G  2.72M   278K 20120313 7 min 31 s
                          how_do_you_know_you_exist_james_zucker  6.00M  5.85G  2.16M   277K 20140814 3 min 2 s
    why_do_honeybees_love_hexagons_zack_patterson_and_andy_peter  8.57M  5.85G  2.15M   275K 20140610 3 min 58 s
             how_to_make_your_writing_funnier_cheri_steinkellner  8.83M  5.86G  1.60M   274K 20160209 5 min 6 s
             why_do_people_fear_the_wrong_things_gerd_gigerenzer  8.68M  5.87G   547K   273K 20200225 4 min 40 s
                                 how_big_is_the_ocean_scott_gass  7.83M  5.88G  2.40M   273K 20130624 5 min 25 s
              the_most_colorful_gemstones_on_earth_jeff_dekofsky  30.6M  5.91G   273K   273K 20201203 5 min 55 s
          the_origin_of_countless_conspiracy_theories_patrickjmt  10.7M  5.92G  1.60M   273K 20160519 4 min 35 s
                         why_do_we_feel_nostalgia_clay_routledge  7.37M  5.93G  1.33M   272K 20161121 4 min 8 s
          why_do_people_get_so_anxious_about_math_orly_rubinsten  6.01M  5.93G  1.33M   271K 20170327 4 min 36 s
            why_should_you_read_charles_dickens_iseult_gillespie  16.6M  5.95G  1.06M   271K 20171221 5 min 16 s
                          history_vs_andrew_jackson_james_fester  7.65M  5.96G  2.12M   271K 20140121 4 min 53 s
                         do_animals_have_language_michele_bishop  7.90M  5.96G  1.59M   271K 20150910 4 min 54 s
      could_the_earth_be_swallowed_by_a_black_hole_fabio_pacucci  16.1M  5.98G   805K   268K 20180920 5 min 10 s
    the_accident_that_changed_the_world_allison_ramsey_and_mary_  7.87M  5.99G   537K   268K 20200210 4 min 50 s
                         the_history_of_tattoos_addison_anderson  10.3M  6.00G  1.83M   268K 20140918 5 min 16 s
    the_myth_of_jason_medea_and_the_golden_fleece_iseult_gillesp  21.0M  6.02G   535K   267K 20200721 4 min 46 s
               dark_matter_the_matter_we_can_t_see_james_gillies  22.0M  6.04G  2.35M   267K 20130503 5 min 34 s
                               how_bones_make_blood_melody_smith  7.43M  6.05G   533K   267K 20200127 4 min 42 s
         the_rise_and_fall_of_the_celtic_warriors_philip_freeman  16.6M  6.06G   533K   267K 20200720 5 min 11 s
    the_silk_road_connecting_the_ancient_world_through_trade_sha  9.08M  6.07G  2.07M   265K 20140603 5 min 19 s
                      what_is_consciousness_michael_s_a_graziano  7.85M  6.08G   794K   265K 20190211 5 min 12 s
               how_does_your_brain_respond_to_pain_karen_d_davis  9.89M  6.09G  2.06M   264K 20140602 4 min 57 s
    how_much_of_human_history_is_on_the_bottom_of_the_ocean_pete  9.33M  6.10G  1.29M   264K 20161020 4 min 45 s
                 a_different_way_to_visualize_rhythm_john_varney  9.61M  6.11G  1.80M   264K 20141020 5 min 22 s
                            does_grammar_matter_andreea_s_calude  7.88M  6.12G  1.54M   263K 20160412 4 min 38 s
        what_happens_when_you_have_a_concussion_clifford_robbins  10.3M  6.13G  1.28M   261K 20170727 6 min 15 s
    from_pacifist_to_spy_wwiis_surprising_secret_agent_shrabani_  9.21M  6.13G   783K   261K 20190806 4 min 30 s
                       why_is_yawning_contagious_claudia_aguirre  10.5M  6.15G  2.03M   260K 20131107 4 min 28 s
        could_we_harness_the_power_of_a_black_hole_fabio_pacucci  23.2M  6.17G   260K   260K 20201019 5 min 41 s
           a_day_in_the_life_of_a_peruvian_shaman_gabriel_prieto  7.63M  6.18G   519K   259K 20200604 4 min 39 s
           what_happens_when_your_dna_is_damaged_monica_menesini  7.65M  6.18G  1.52M   259K 20150921 4 min 58 s
    what_can_dna_tests_really_tell_us_about_our_ancestry_prosant  21.9M  6.20G   517K   259K 20200609 6 min 44 s
             what_are_the_universal_human_rights_benedetta_berti  12.0M  6.22G  1.51M   258K 20151015 4 min 46 s
             how_do_dogs_see_with_their_noses_alexandra_horowitz  10.1M  6.23G  1.77M   258K 20150202 4 min 27 s
              what_do_all_languages_have_in_common_cameron_morin  14.9M  6.24G   516K   258K 20200629 5 min 18 s
                           michio_kaku_what_is_deja_vu_big_think  6.56M  6.25G  2.51M   257K 20111115 3 min 0 s
    three_anti_social_skills_to_improve_your_writing_nadia_kalma  6.89M  6.25G  2.26M   257K 20121120 3 min 44 s
                       what_is_metallic_glass_ashwini_bharathula  6.04M  6.26G  1.49M   255K 20160317 4 min 33 s
    a_day_in_the_life_of_an_ancient_greek_architect_mark_robinso  20.3M  6.28G   253K   253K 20200915 5 min 33 s
        the_breathtaking_courage_of_harriet_tubman_janell_hobson  8.17M  6.29G  0.99M   252K 20180724 4 min 48 s
                            is_radiation_dangerous_matt_anticole  9.42M  6.30G  1.48M   252K 20160314 5 min 20 s
    should_you_trust_your_first_impression_peter_mende_siedlecki  9.92M  6.31G  2.22M   252K 20130815 4 min 38 s
                               what_does_the_liver_do_emma_bryce  5.81M  6.31G  1.72M   252K 20141125 3 min 24 s
                     volcanic_eruption_explained_steven_anderson  21.7M  6.33G   504K   252K 20200713 5 min 33 s
    how_did_polynesian_wayfinders_navigate_the_pacific_ocean_ala  18.1M  6.35G  0.98M   252K 20171017 5 min 31 s
      the_last_living_members_of_an_extinct_species_jan_stejskal  16.3M  6.37G   503K   252K 20200813 5 min 31 s
    one_of_the_most_epic_engineering_feats_in_history_alex_gendl  17.9M  6.38G   503K   252K 20200211 5 min 15 s
                   what_s_so_special_about_viking_ships_jan_bill  13.0M  6.40G   501K   251K 20200121 4 min 57 s
                          why_are_sharks_so_awesome_tierney_thys  12.3M  6.41G  1.22M   250K 20161107 4 min 35 s
         why_should_you_read_kafka_on_the_shore_iseult_gillespie  14.2M  6.42G   745K   248K 20190820 4 min 40 s
              the_science_of_static_electricity_anuradha_bhagwat  6.29M  6.43G  1.69M   248K 20150409 3 min 38 s
    how_is_power_divided_in_the_united_states_government_belinda  5.20M  6.43G  2.17M   247K 20130412 3 min 49 s
    how_the_world_s_first_metro_system_was_built_christian_wolma  8.97M  6.44G   988K   247K 20180419 4 min 57 s
                                how_batteries_work_adam_jacobson  6.38M  6.45G  1.68M   246K 20150521 4 min 19 s
           myths_and_misconceptions_about_evolution_alex_gendler  7.18M  6.46G  2.16M   246K 20130708 4 min 22 s
    the_myth_of_ireland_s_two_greatest_warriors_iseult_gillespie  18.0M  6.47G   490K   245K 20200806 4 min 47 s
                                    how_do_lungs_work_emma_bryce  5.98M  6.48G  1.67M   244K 20141124 3 min 21 s
                          can_we_eat_to_starve_cancer_william_li  35.4M  6.51G  1.90M   244K 20140408 20 min 1 s
    how_can_we_solve_the_antibiotic_resistance_crisis_gerry_wrig  11.9M  6.53G   487K   243K 20200316 6 min 22 s
        the_psychology_behind_irrational_decisions_sara_garofalo  6.80M  6.53G  1.42M   242K 20160512 4 min 38 s
              newtons_three_body_problem_explained_fabio_pacucci  17.6M  6.55G   484K   242K 20200803 5 min 30 s
               the_complicated_history_of_surfing_scott_laderman  14.1M  6.56G   967K   242K 20171116 5 min 39 s
           the_race_to_decode_a_mysterious_language_susan_lupack  15.6M  6.58G   483K   241K 20200714 4 min 44 s
               why_should_you_read_sylvia_plath_iseult_gillespie  10.9M  6.59G   723K   241K 20190307 4 min 45 s
    whats_the_smallest_thing_in_the_universe_jonathan_butterwort  10.2M  6.60G   722K   241K 20181115 5 min 20 s
            what_we_know_and_don_t_know_about_ebola_alex_gendler  8.20M  6.61G  1.64M   239K 20141204 4 min 0 s
                                       what_is_fat_george_zaidan  7.34M  6.61G  2.09M   238K 20130522 4 min 21 s
    the_akune_brothers_siblings_on_opposite_sides_of_war_wendell  12.8M  6.63G  1.62M   237K 20150721 4 min 53 s
                    is_math_discovered_or_invented_jeff_dekofsky  14.1M  6.64G  1.61M   235K 20141027 5 min 10 s
           ugly_history_the_1937_haitian_massacre_edward_paulino  17.8M  6.66G   940K   235K 20180125 5 min 39 s
       can_you_solve_the_mondrian_squares_riddle_gordon_hamilton  5.64M  6.66G   940K   235K 20180628 4 min 45 s
                          the_resistance_think_like_a_coder_ep_2  11.9M  6.67G   470K   235K 20191014 6 min 9 s
                               who_owns_the_wilderness_elyse_cox  20.1M  6.69G   234K   234K 20201006 5 min 12 s
                     why_should_you_read_hamlet_iseult_gillespie  10.8M  6.70G   697K   232K 20190625 5 min 8 s
    the_psychology_of_post_traumatic_stress_disorder_joelle_rabo  22.0M  6.73G   930K   232K 20180625 5 min 12 s
    population_pyramids_powerful_predictors_of_the_future_kim_pr  10.9M  6.74G  1.81M   232K 20140505 5 min 1 s
    group_theory_101_how_to_play_a_rubiks_cube_like_a_piano_mich  7.58M  6.74G  1.36M   232K 20151102 4 min 36 s
    how_in_vitro_fertilization_ivf_works_nassim_assefi_and_brian  14.5M  6.76G  1.58M   231K 20150507 6 min 42 s
                      how_do_blood_transfusions_work_bill_schutt  14.4M  6.77G   462K   231K 20200218 4 min 51 s
    why_should_you_read_the_god_of_small_things_by_arundhati_roy  11.6M  6.78G   462K   231K 20190924 4 min 31 s
        is_there_a_disease_that_makes_us_love_cats_jaap_de_roode  9.03M  6.79G  1.35M   231K 20160623 5 min 5 s
                           why_do_our_bodies_age_monica_menesini  8.17M  6.80G  1.35M   231K 20160609 5 min 9 s
    the_spear_wielding_stork_who_revolutionized_science_lucy_coo  20.7M  6.82G   231K   231K 20201217 5 min 41 s
                   making_sense_of_irrational_numbers_ganesh_pai  5.69M  6.83G  1.35M   231K 20160523 4 min 40 s
                the_secrets_of_mozarts_magic_flute_joshua_borths  7.25M  6.83G  1.12M   230K 20161122 5 min 47 s
                                  the_science_of_skin_emma_bryce  7.01M  6.84G   917K   229K 20180312 5 min 10 s
                            how_do_your_hormones_work_emma_bryce  9.95M  6.85G   913K   228K 20180621 5 min 3 s
                    could_we_actually_live_on_mars_mari_foroutan  8.45M  6.86G  1.56M   228K 20150824 4 min 29 s
    everything_you_need_to_know_to_read_homer_s_odyssey_jill_das  10.2M  6.87G  1.11M   228K 20170130 4 min 56 s
                                      comma_story_terisa_folaron  7.47M  6.88G  2.01M   228K 20130709 4 min 59 s
                    how_do_executive_orders_work_christina_greer  8.88M  6.88G   911K   228K 20170918 4 min 46 s
    the_difference_between_classical_and_operant_conditioning_pe  6.51M  6.89G  2.00M   228K 20130307 4 min 12 s
                             what_is_dust_made_of_michael_marder  10.1M  6.90G   908K   227K 20180524 4 min 33 s
                      where_do_new_words_come_from_marcel_danesi  9.27M  6.91G   907K   227K 20170907 5 min 43 s
                                   the_truth_about_bats_amy_wray  15.6M  6.92G  1.55M   227K 20141216 5 min 47 s
    earworms_those_songs_that_get_stuck_in_your_head_elizabeth_h  8.60M  6.93G  1.55M   226K 20150326 4 min 45 s
                               think_like_a_coder_teaser_trailer  2.39M  6.94G   450K   225K 20190921 56 s 63 ms
                    how_does_hibernation_work_sheena_lee_faherty  9.83M  6.95G   891K   223K 20180503 4 min 48 s
        the_colossal_consequences_of_supervolcanoes_alex_gendler  12.1M  6.96G  1.74M   222K 20140609 4 min 50 s
    ted_ed_is_on_patreon_we_need_your_help_to_revolutionize_educ  5.87M  6.96G  1.08M   222K 20170815 1 min 51 s
                               inside_your_computer_bettina_bair  6.95M  6.97G  1.94M   221K 20130701 4 min 11 s
                 if_superpowers_were_real_super_strength_joy_lin  12.9M  6.98G  1.94M   221K 20130628 4 min 4 s
    getting_started_as_a_dj_mixing_mashups_and_digital_turntable  32.6M  7.01G  1.73M   221K 20140303 10 min 36 s
        how_does_cancer_spread_through_the_body_ivan_seah_yu_jun  9.36M  7.02G  1.51M   220K 20141120 4 min 43 s
                                     why_do_we_sweat_john_murnan  10.3M  7.03G   881K   220K 20180515 4 min 47 s
                  the_brilliance_of_bioluminescence_leslie_kenna  6.52M  7.04G  1.91M   218K 20130515 4 min 8 s
                         could_we_create_dark_matter_rolf_landua  11.2M  7.05G  1.06M   217K 20170817 5 min 48 s
    an_athlete_uses_physics_to_shatter_world_records_asaf_bar_yo  5.59M  7.06G  1.69M   217K 20140218 3 min 50 s
                       a_quantum_thought_experiment_matteo_fadel  20.4M  7.08G   216K   216K 20201214 4 min 58 s
          master_the_art_of_public_speaking_with_ted_masterclass  5.27M  7.08G   432K   216K 20200104 1 min 40 s
    first_person_vs_second_person_vs_third_person_rebekah_bergma  18.3M  7.10G   430K   215K 20200625 5 min 19 s
                         if_superpowers_were_real_flight_joy_lin  20.6M  7.12G  1.87M   212K 20130628 5 min 10 s
               how_turtle_shells_evolved_twice_judy_cebra_thomas  9.08M  7.13G   637K   212K 20190730 4 min 45 s
            the_ethical_dilemma_of_self_driving_cars_patrick_lin  14.7M  7.14G  1.24M   212K 20151208 4 min 15 s
                                    what_is_a_vector_david_huynh  5.96M  7.15G  1.03M   212K 20160913 4 min 40 s
    why_certain_naturally_occurring_wildfires_are_necessary_jim_  8.56M  7.16G  1.24M   212K 20160202 4 min 20 s
                                   rhapsody_on_the_proof_of_pi_4  21.3M  7.18G  2.06M   211K 20120405 5 min 54 s
                    the_threat_of_invasive_species_jennifer_klos  8.62M  7.19G  1.24M   211K 20160503 4 min 45 s
    does_the_wonderful_wizard_of_oz_have_a_hidden_message_david_  8.28M  7.19G  1.03M   211K 20170306 4 min 43 s
                       when_is_water_safe_to_drink_mia_nacamulli  11.6M  7.20G  1.03M   211K 20170807 5 min 23 s
      nature_s_smallest_factory_the_calvin_cycle_cathy_symington  9.70M  7.21G  1.64M   210K 20140401 5 min 37 s
            should_we_get_rid_of_standardized_testing_arlo_kempf  10.2M  7.22G   841K   210K 20170919 5 min 40 s
                    da_vinci_s_vitruvian_man_of_math_james_earle  6.30M  7.23G  1.85M   210K 20130711 3 min 20 s
    situational_irony_the_opposite_of_what_you_think_christopher  5.88M  7.24G  1.84M   210K 20121213 3 min 11 s
    this_one_weird_trick_will_help_you_spot_clickbait_jeff_leek_  8.37M  7.24G   627K   209K 20190606 5 min 37 s
    what_can_schrodinger_s_cat_teach_us_about_quantum_mechanics_  16.4M  7.26G  1.63M   209K 20140821 5 min 23 s
                urbanization_and_the_future_of_cities_vance_kite  8.36M  7.27G  1.62M   208K 20130912 4 min 8 s
    is_it_possible_to_create_a_perfect_vacuum_rolf_landua_and_an  18.3M  7.29G   828K   207K 20170912 4 min 31 s
      the_fascinating_science_behind_phantom_limbs_joshua_w_pate  12.2M  7.30G   620K   207K 20181004 5 min 14 s
    when_will_the_next_mass_extinction_occur_borths_d_emic_and_p  12.7M  7.31G  1.21M   206K 20160121 5 min 0 s
                how_do_you_know_if_you_have_a_virus_cella_wright  10.7M  7.32G   412K   206K 20200518 5 min 4 s
    the_hidden_network_that_makes_the_internet_possible_sajan_sa  12.9M  7.33G   617K   206K 20190422 5 min 19 s
    performing_brain_surgery_without_a_scalpel_hyunsoo_joshua_no  17.4M  7.35G   205K   205K 20200928 5 min 17 s
               how_close_are_we_to_eradicating_hiv_philip_a_chan  11.3M  7.36G   614K   205K 20190610 4 min 53 s
                      the_fish_that_walk_on_land_noah_r_bressman  17.7M  7.38G   409K   205K 20200831 5 min 46 s
    the_moral_roots_of_liberals_and_conservatives_jonathan_haidt  35.6M  7.41G  1.80M   205K 20121231 18 min 39 s
    why_are_earthquakes_so_hard_to_predict_jean_baptiste_p_koehl  9.91M  7.42G   611K   204K 20190408 4 min 53 s
                          inside_the_ant_colony_deborah_m_gordon  12.4M  7.44G  1.59M   204K 20140708 4 min 46 s
             the_genius_of_mendeleev_s_periodic_table_lou_serico  7.03M  7.44G  1.79M   204K 20121121 4 min 24 s
            what_happens_when_continents_collide_juan_d_carrillo  13.3M  7.46G  1.39M   203K 20150818 4 min 57 s
                             how_do_contraceptives_work_nwhunter  6.60M  7.46G  0.99M   202K 20160919 4 min 20 s
                   the_first_and_last_king_of_haiti_marlene_daut  8.84M  7.47G   405K   202K 20191007 5 min 10 s
    how_playing_sports_benefits_your_body_and_your_brain_leah_la  8.94M  7.48G  1.18M   202K 20160628 3 min 46 s
        how_magellan_circumnavigated_the_globe_ewandro_magalhaes  15.8M  7.49G  0.98M   202K 20170330 5 min 52 s
                  how_fast_are_you_moving_right_now_tucker_hiatt  16.2M  7.51G  1.57M   201K 20140127 6 min 9 s
    the_chaotic_brilliance_of_artist_jean_michel_basquiat_jordan  11.5M  7.52G   602K   201K 20190228 4 min 32 s
    the_last_banana_a_thought_experiment_in_probability_leonardo  8.23M  7.53G  1.37M   200K 20150223 4 min 9 s
    everything_you_need_to_know_to_read_frankenstein_iseult_gill  9.18M  7.54G   995K   199K 20170223 5 min 1 s
    what_is_phantom_traffic_and_why_is_it_ruining_your_life_benj  10.0M  7.55G   398K   199K 20200528 4 min 53 s
    how_interpreters_juggle_two_languages_at_once_ewandro_magalh  15.2M  7.56G  1.16M   197K 20160607 4 min 55 s
             the_ancient_origins_of_the_olympics_armand_d_angour  9.25M  7.57G  1.35M   197K 20150903 3 min 19 s
    whats_the_difference_between_a_scientific_law_and_theory_mat  9.55M  7.58G  1.15M   197K 20151119 5 min 11 s
    the_romans_flooded_the_colosseum_for_sea_battles_janelle_pet  7.65M  7.59G   590K   197K 20190624 4 min 19 s
                    why_should_you_read_moby_dick_sascha_morrell  13.7M  7.60G   393K   196K 20200526 5 min 58 s
    how_one_journalist_risked_her_life_to_hold_murderers_account  8.26M  7.61G   588K   196K 20190204 4 min 49 s
                               how_to_use_a_semicolon_emma_bryce  5.84M  7.62G  1.34M   196K 20150706 3 min 35 s
    whats_so_great_about_the_great_lakes_cheri_dobbs_and_jennife  10.8M  7.63G   977K   195K 20170110 4 min 46 s
                      what_happens_when_you_die_a_poetic_inquiry  11.0M  7.64G   195K   195K 20201215 2 min 29 s
                      the_rise_of_modern_populism_takis_s_pappas  23.5M  7.66G   389K   195K 20200820 6 min 21 s
    everything_you_need_to_know_to_read_the_canterbury_tales_ise  8.15M  7.67G   583K   194K 20181002 4 min 35 s
                   can_the_ocean_run_out_of_oxygen_kate_slabosky  26.4M  7.69G   387K   193K 20200818 6 min 20 s
    who_s_at_risk_for_colon_cancer_amit_h_sachdev_and_frank_g_gr  7.69M  7.70G   772K   193K 20180104 4 min 43 s
    notes_of_a_native_son_the_world_according_to_james_baldwin_c  7.73M  7.71G   577K   192K 20190212 4 min 13 s
        it_s_a_church_it_s_a_mosque_it_s_hagia_sophia_kelly_wall  14.3M  7.72G  1.50M   192K 20140714 5 min 11 s
                      the_world_machine_think_like_a_coder_ep_10  54.9M  7.78G   191K   191K 20200924 12 min 9 s
           are_we_running_out_of_clean_water_balsher_singh_sidhu  16.5M  7.79G   573K   191K 20181206 5 min 18 s
          why_should_you_read_waiting_for_godot_iseult_gillespie  10.1M  7.80G   573K   191K 20181015 5 min 3 s
     why_do_you_need_to_get_a_flu_shot_every_year_melvin_sanicas  8.94M  7.81G   761K   190K 20171120 5 min 11 s
         prohibition_banning_alcohol_was_a_bad_idea_rod_phillips  20.2M  7.83G   380K   190K 20200709 5 min 12 s
                           why_do_whales_sing_stephanie_sardelis  9.43M  7.84G   950K   190K 20161110 5 min 12 s
                  the_2400_year_search_for_the_atom_theresa_doud  8.42M  7.85G  1.30M   189K 20141208 5 min 22 s
    how_miscommunication_happens_and_how_to_avoid_it_katherine_h  8.33M  7.86G  1.11M   189K 20160222 4 min 32 s
     vermicomposting_how_worms_can_reduce_our_waste_matthew_ross  6.59M  7.86G  1.66M   188K 20130626 4 min 29 s
               why_is_this_painting_so_shocking_iseult_gillespie  10.2M  7.87G   564K   188K 20190430 5 min 15 s
                        how_x_rays_see_through_your_skin_ge_wang  8.37M  7.88G  1.28M   188K 20150622 4 min 42 s
    how_the_konigsberg_bridge_problem_changed_mathematics_dan_va  7.79M  7.89G  1.10M   188K 20160901 4 min 38 s
                                          insults_by_shakespeare  12.9M  7.90G  1.83M   187K 20120504 6 min 23 s
    why_havent_we_cured_arthritis_kaitlyn_sadtler_and_heather_j_  6.52M  7.91G   375K   187K 20191107 4 min 24 s
                   where_did_earths_water_come_from_zachary_metz  6.17M  7.91G  1.28M   187K 20150323 3 min 52 s
     how_big_is_a_mole_not_the_animal_the_other_one_daniel_dulek  8.84M  7.92G  1.65M   187K 20120911 4 min 32 s
                why_do_buildings_fall_in_earthquakes_vicki_v_may  11.3M  7.93G  1.27M   186K 20150126 4 min 51 s
                  grammar_s_great_divide_the_oxford_comma_ted_ed  6.93M  7.94G  1.46M   186K 20140317 3 min 25 s
                    if_superpowers_were_real_super_speed_joy_lin  17.0M  7.96G  1.64M   186K 20130628 4 min 50 s
         you_are_your_microbes_jessica_green_and_karen_guillemin  8.29M  7.97G  1.63M   186K 20130107 3 min 45 s
         the_greatest_machine_that_never_was_john_graham_cumming  21.4M  7.99G  1.63M   185K 20130619 12 min 14 s
                 why_people_fall_for_misinformation_joseph_isaac  19.2M  8.00G   370K   185K 20200903 5 min 15 s
               are_food_preservatives_bad_for_you_eleanor_nelsen  7.47M  8.01G   919K   184K 20161108 4 min 52 s
    how_this_disease_changes_the_shape_of_your_cells_amber_m_yat  11.2M  8.02G   549K   183K 20190506 4 min 40 s
    the_secret_language_of_trees_camille_defrenne_and_suzanne_si  8.80M  8.03G   548K   183K 20190701 4 min 33 s
                     what_are_gravitational_waves_amber_l_stuver  7.34M  8.04G   728K   182K 20170914 5 min 24 s
                        the_furnace_bots_think_like_a_coder_ep_3  11.3M  8.05G   361K   180K 20191118 6 min 11 s
             how_to_biohack_your_cells_to_fight_cancer_greg_foot  24.5M  8.07G   540K   180K 20190409 8 min 0 s
                       how_do_geckos_defy_gravity_eleanor_nelsen  7.93M  8.08G  1.23M   179K 20150330 4 min 29 s
    how_exactly_does_binary_code_work_jose_americo_n_l_f_de_frei  8.37M  8.09G   714K   178K 20180712 4 min 39 s
                                  why_the_solar_system_can_exist  3.58M  8.09G  1.73M   177K 20120503 2 min 6 s
                          why_is_being_scared_so_fun_margee_kerr  10.8M  8.10G  1.04M   177K 20160421 4 min 28 s
            what_happens_when_you_get_heat_stroke_douglas_j_casa  6.07M  8.11G  1.38M   176K 20140721 3 min 53 s
      how_small_are_we_in_the_scale_of_the_universe_alex_hofeldt  9.16M  8.12G   876K   175K 20170213 4 min 7 s
                     why_are_human_bodies_asymmetrical_leo_q_wan  8.05M  8.13G  1.02M   175K 20160125 4 min 18 s
                     the_law_of_conservation_of_mass_todd_ramsey  8.86M  8.14G  1.19M   174K 20150226 4 min 36 s
                                    eye_vs_camera_michael_mauser  13.3M  8.15G  1.19M   174K 20150406 4 min 56 s
                        a_3d_atlas_of_the_universe_carter_emmart  18.8M  8.17G  1.53M   174K 20130308 6 min 57 s
                               what_s_an_algorithm_david_j_malan  7.77M  8.17G  1.52M   173K 20130520 4 min 57 s
      whats_a_squillo_and_why_do_opera_singers_need_it_ming_luke  16.3M  8.19G   347K   173K 20200309 5 min 12 s
    how_many_ways_can_you_arrange_a_deck_of_cards_yannay_khaikin  5.19M  8.20G  1.35M   173K 20140327 3 min 41 s
            the_secret_messages_of_viking_runestones_jesse_byock  6.43M  8.20G   344K   172K 20200220 4 min 34 s
    the_math_behind_michael_jordans_legendary_hang_time_andy_pet  6.10M  8.21G  1.17M   172K 20150604 3 min 45 s
                         the_train_heist_think_like_a_coder_ep_4  14.6M  8.22G   342K   171K 20191209 5 min 59 s
             you_are_more_transparent_than_you_think_sajan_saini  9.69M  8.23G   513K   171K 20190604 5 min 22 s
                   inside_the_minds_of_animals_bryan_b_rasmussen  9.40M  8.24G  1.17M   171K 20150714 5 min 12 s
                                     the_secret_life_of_plankton  18.3M  8.26G  1.66M   170K 20120402 6 min 1 s
                     how_do_drugs_affect_the_brain_sara_garofalo  9.52M  8.27G   851K   170K 20170629 5 min 4 s
                           how_false_news_can_spread_noah_tavlin  6.63M  8.27G  1.16M   170K 20150827 3 min 41 s
                      evolutions_great_mystery_michael_corballis  17.5M  8.29G   340K   170K 20200824 4 min 42 s
    the_turing_test_can_a_computer_pass_for_a_human_alex_gendler  11.1M  8.30G  0.99M   170K 20160425 4 min 42 s
                                    how_do_we_smell_rose_eveleth  8.22M  8.31G  1.32M   169K 20131219 4 min 19 s
    the_basics_of_the_higgs_boson_dave_barney_and_steve_goldfarb  8.81M  8.32G  1.48M   169K 20130503 6 min 29 s
    will_the_ocean_ever_run_out_of_fish_ayana_elizabeth_johnson_  8.77M  8.33G   843K   169K 20170810 4 min 27 s
                      how_do_animals_see_in_the_dark_anna_stockl  12.7M  8.34G  0.98M   168K 20160825 4 min 22 s
                 a_brief_history_of_melancholy_courtney_stephens  8.22M  8.35G  1.15M   168K 20141002 5 min 28 s
                     how_crispr_lets_you_edit_dna_andrea_m_henle  8.32M  8.36G   502K   167K 20190124 5 min 28 s
                  lets_plant_20_million_trees_together_teamtrees  2.39M  8.36G   332K   166K 20191025 1 min 0 s
       is_the_weather_actually_becoming_more_extreme_r_saravanan  20.6M  8.38G   331K   166K 20200825 5 min 21 s
        this_sea_creature_breathes_through_its_butt_cella_wright  11.5M  8.39G   330K   165K 20200430 5 min 2 s
             the_wildly_complex_anatomy_of_a_sneaker_angel_chang  9.13M  8.40G   330K   165K 20200423 5 min 22 s
                     what_is_epigenetics_carlos_guerrero_bosagna  11.0M  8.41G   989K   165K 20160627 5 min 2 s
      would_winning_the_lottery_make_you_happier_raj_raghunathan  7.85M  8.42G   823K   165K 20170207 4 min 35 s
    the_truth_about_electroconvulsive_therapy_ect_helen_m_farrel  8.03M  8.42G   494K   165K 20190114 4 min 23 s
                    do_politics_make_us_irrational_jay_van_bavel  13.2M  8.44G   329K   165K 20200204 5 min 36 s
              why_are_there_so_many_types_of_apples_theresa_doud  6.49M  8.44G   822K   164K 20160922 4 min 27 s
    how_do_cancer_cells_behave_differently_from_healthy_ones_geo  11.7M  8.46G  1.44M   164K 20121205 3 min 50 s
    what_you_might_not_know_about_the_declaration_of_independenc  7.85M  8.46G  1.28M   164K 20140701 3 min 37 s
      the_mysterious_origins_of_life_on_earth_luka_seamus_wright  11.0M  8.47G   492K   164K 20190826 4 min 56 s
                     how_many_universes_are_there_chris_anderson  15.2M  8.49G  1.60M   164K 20120311 4 min 42 s
                 the_ballet_that_incited_a_riot_iseult_gillespie  11.4M  8.50G   326K   163K 20200114 5 min 2 s
    aphasia_the_disorder_that_makes_you_lose_your_words_susan_wo  14.1M  8.51G   814K   163K 20160915 5 min 10 s
     the_beneficial_bacteria_that_make_delicious_food_erez_garty  7.20M  8.52G   976K   163K 20160119 4 min 39 s
                               how_menstruation_works_emma_bryce  8.57M  8.53G   976K   163K 20160112 4 min 11 s
                                  the_survival_of_the_sea_turtle  7.90M  8.54G  1.59M   163K 20120723 4 min 25 s
    how_close_are_we_to_uploading_our_minds_michael_s_a_graziano  8.59M  8.54G   325K   163K 20191028 5 min 5 s
            who_was_the_world_s_first_author_soraya_field_fiorio  9.72M  8.55G   325K   163K 20200323 4 min 55 s
    the_history_of_the_barometer_and_how_it_works_asaf_bar_yosef  6.07M  8.56G  1.27M   162K 20140728 4 min 45 s
                             how_do_your_kidneys_work_emma_bryce  6.69M  8.57G  1.11M   162K 20150209 3 min 54 s
                         what_is_verbal_irony_christopher_warner  8.04M  8.57G  1.42M   161K 20130313 3 min 28 s
    why_shakespeare_loved_iambic_pentameter_david_t_freeman_and_  10.3M  8.58G  1.10M   161K 20150127 5 min 21 s
                    does_stress_affect_your_memory_elizabeth_cox  8.10M  8.59G   643K   161K 20180904 4 min 43 s
                                       what_is_love_brad_troeger  14.7M  8.61G  1.25M   160K 20130909 4 min 59 s
    how_to_speed_up_chemical_reactions_and_get_a_date_aaron_sams  11.1M  8.62G  1.56M   160K 20120618 4 min 55 s
               why_should_you_read_virgil_s_aeneid_mark_robinson  17.1M  8.63G   636K   159K 20171019 5 min 35 s
                      why_do_we_kiss_under_mistletoe_carlos_reif  6.59M  8.64G   792K   158K 20161222 4 min 41 s
                   why_do_we_have_to_wear_sunscreen_kevin_p_boyd  8.69M  8.65G  1.39M   158K 20130806 5 min 1 s
       how_does_your_body_know_what_time_it_is_marco_a_sotomayor  7.91M  8.66G   790K   158K 20161208 5 min 8 s
                         what_happened_to_antimatter_rolf_landua  14.4M  8.67G  1.39M   158K 20130503 5 min 16 s
                     the_mystery_of_motion_sickness_rose_eveleth  5.10M  8.68G  1.23M   158K 20140113 3 min 9 s
    why_is_this_painting_so_captivating_james_earle_and_christin  6.89M  8.68G   943K   157K 20160310 3 min 52 s
                    the_evolution_of_the_human_eye_joshua_harvey  7.25M  8.69G  1.07M   156K 20150108 4 min 43 s
                          secrets_of_the_x_chromosome_robin_ball  6.36M  8.70G   779K   156K 20170418 5 min 5 s
             a_day_in_the_life_of_a_cossack_warrior_alex_gendler  14.2M  8.71G   467K   156K 20190822 4 min 47 s
        how_many_verb_tenses_are_there_in_english_anna_ananichuk  9.51M  8.72G   623K   156K 20171106 4 min 27 s
                the_hidden_life_of_rosa_parks_riche_d_richardson  11.7M  8.73G   307K   154K 20200413 4 min 59 s
    why_should_you_read_a_midsummer_night_s_dream_iseult_gillesp  13.0M  8.74G   461K   154K 20181203 4 min 42 s
        einstein_s_brilliant_mistake_entangled_states_chad_orzel  8.45M  8.75G  1.05M   153K 20141016 5 min 9 s
                            who_was_confucius_bryan_w_van_norden  10.6M  8.76G   917K   153K 20151027 4 min 29 s
                         whats_a_smartphone_made_of_kim_preshoff  10.9M  8.77G   458K   153K 20181001 4 min 55 s
       why_should_you_read_the_master_and_margarita_alex_gendler  7.97M  8.78G   456K   152K 20190530 4 min 32 s
    why_does_ice_float_in_water_george_zaidan_and_charles_morton  8.04M  8.79G  1.18M   151K 20131022 3 min 55 s
                   how_statistics_can_be_misleading_mark_liddell  10.0M  8.80G   908K   151K 20160114 4 min 18 s
         why_should_you_read_midnights_children_iseult_gillespie  19.5M  8.82G   300K   150K 20190910 5 min 55 s
                   how_long_will_human_impacts_last_david_biello  8.98M  8.83G   598K   149K 20171204 5 min 29 s
    what_color_is_tuesday_exploring_synesthesia_richard_e_cytowi  9.95M  8.84G  1.30M   148K 20130610 3 min 56 s
                 how_do_our_brains_process_speech_gareth_gaskell  23.4M  8.86G   296K   148K 20200723 4 min 53 s
    the_big_beaked_rock_munching_fish_that_protect_coral_reefs_m  24.2M  8.88G   294K   147K 20200804 5 min 4 s
    why_is_nasa_sending_a_spacecraft_to_a_metal_world_linda_t_el  8.76M  8.89G   584K   146K 20180129 4 min 19 s
                   if_superpowers_were_real_invisibility_joy_lin  13.4M  8.90G  1.28M   146K 20130627 4 min 32 s
    how_the_sandwich_was_invented_moments_of_vision_5_jessica_or  3.90M  8.91G   729K   146K 20161103 1 min 48 s
                              how_did_teeth_evolve_peter_s_ungar  12.0M  8.92G   583K   146K 20180205 4 min 44 s
    the_fundamentals_of_space_time_part_1_andrew_pontzen_and_tom  7.99M  8.93G  1.13M   145K 20140324 5 min 5 s
                               the_chasm_think_like_a_coder_ep_6  16.3M  8.94G   288K   144K 20200130 6 min 40 s
                          the_death_of_the_universe_renee_hlozek  6.52M  8.95G  1.12M   144K 20131212 4 min 38 s
                         can_animals_be_deceptive_eldridge_adams  7.42M  8.96G   429K   143K 20181218 4 min 55 s
                                        introducing_earth_school  2.24M  8.96G   286K   143K 20200422 54 s 450 ms
         a_3_minute_guide_to_the_bill_of_rights_belinda_stutzman  7.50M  8.97G  1.25M   143K 20121030 3 min 34 s
                      if_superpowers_were_real_body_mass_joy_lin  21.6M  8.99G  1.25M   143K 20130627 6 min 44 s
                   the_tower_of_epiphany_think_like_a_coder_ep_7  19.5M  9.01G   285K   143K 20200227 8 min 13 s
             the_left_brain_vs_right_brain_myth_elizabeth_waters  12.2M  9.02G   712K   142K 20170724 4 min 11 s
      how_far_would_you_have_to_go_to_escape_gravity_rene_laufer  10.8M  9.03G   426K   142K 20181106 4 min 55 s
    why_is_aristophanes_called_the_father_of_comedy_mark_robinso  11.0M  9.04G   566K   142K 20180821 5 min 11 s
                                the_science_of_smog_kim_preshoff  8.11M  9.05G   701K   140K 20170831 5 min 43 s
    how_to_see_more_and_care_less_the_art_of_georgia_o_keeffe_is  18.0M  9.06G   279K   139K 20200608 4 min 59 s
    what_does_it_mean_to_be_a_refugee_benedetta_berti_and_evelie  8.78M  9.07G   833K   139K 20160616 5 min 42 s
                                                        caffeine  12.8M  9.09G  1.36M   139K 20120327 4 min 30 s
                             the_coin_flip_conundrum_po_shen_loh  7.01M  9.09G   555K   139K 20180215 4 min 22 s
    what_is_hpv_and_how_can_you_protect_yourself_from_it_emma_br  5.88M  9.10G   416K   139K 20190709 4 min 27 s
                                             one_is_one_or_is_it  5.82M  9.10G  1.35M   138K 20120521 3 min 54 s
                    how_mucus_keeps_us_healthy_katharina_ribbeck  7.94M  9.11G   825K   138K 20151105 4 min 7 s
    there_may_be_extraterrestrial_life_in_our_solar_system_augus  12.8M  9.12G   411K   137K 20190620 5 min 16 s
    the_science_of_stage_fright_and_how_to_overcome_it_mikael_ch  11.4M  9.14G  1.07M   137K 20131008 4 min 7 s
                                   ted_ed_youtube_channel_teaser  4.46M  9.14G  1.20M   137K 20130402 1 min 26 s
               working_backward_to_solve_problems_maurice_ashley  13.9M  9.15G  1.20M   137K 20130311 5 min 56 s
                                what_are_stem_cells_craig_a_kohn  6.30M  9.16G  1.07M   136K 20130910 4 min 10 s
                    the_sexual_deception_of_orchids_anne_gaskett  11.7M  9.17G   408K   136K 20190214 5 min 24 s
                             the_artists_think_like_a_coder_ep_5  14.2M  9.18G   270K   135K 20200113 6 min 41 s
               who_built_great_zimbabwe_and_why_breeanna_elliott  8.88M  9.19G   673K   135K 20170622 5 min 6 s
                            the_gauntlet_think_like_a_coder_ep_8  20.3M  9.21G   269K   134K 20200416 8 min 16 s
                              who_won_the_space_race_jeff_steers  9.93M  9.22G  1.18M   134K 20130814 4 min 46 s
                             an_anti_hero_of_one_s_own_tim_adams  9.86M  9.23G  1.18M   134K 20121113 4 min 10 s
    the_origins_of_ballet_jennifer_tortorello_and_adrienne_westw  8.08M  9.24G   802K   134K 20160307 4 min 37 s
    why_should_you_read_shakespeares_the_tempest_iseult_gillespi  12.0M  9.25G   400K   133K 20190205 4 min 57 s
                                  how_to_grow_a_bone_nina_tandon  7.72M  9.26G   919K   131K 20150625 4 min 36 s
    the_past_present_and_future_of_the_bubonic_plague_sharon_n_d  6.58M  9.27G  1.03M   131K 20140818 4 min 12 s
                      how_to_spot_a_counterfeit_bill_tien_nguyen  8.01M  9.27G   914K   131K 20150416 4 min 5 s
                                  how_to_set_the_table_anna_post  4.52M  9.28G  1.14M   130K 20130626 3 min 26 s
                        who_decides_what_art_means_hayley_levitt  6.96M  9.29G   391K   130K 20181126 4 min 18 s
                                 how_do_scars_form_sarthak_sinha  6.02M  9.29G   910K   130K 20141111 3 min 41 s
                         the_science_of_hearing_douglas_l_oliver  8.76M  9.30G   519K   130K 20180619 5 min 16 s
           the_mathematics_of_sidewalk_illusions_fumiko_futamura  13.4M  9.31G   649K   130K 20170123 4 min 54 s
          how_optical_illusions_trick_your_brain_nathan_s_jacobs  13.3M  9.33G  1.01M   130K 20140812 5 min 18 s
                           whats_the_point_e_of_ballet_ming_luke  13.0M  9.34G   259K   129K 20200420 5 min 9 s
                          real_life_sunken_cities_peter_campbell  10.0M  9.35G   775K   129K 20160804 4 min 30 s
                    what_is_alzheimer_s_disease_ivan_seah_yu_jun  8.69M  9.36G  1.01M   129K 20140403 3 min 49 s
                  the_lovable_and_lethal_sea_lion_claire_simeone  8.83M  9.37G   385K   128K 20190528 4 min 36 s
            a_brief_history_of_numerical_systems_alessandra_king  6.81M  9.37G   642K   128K 20170119 5 min 7 s
         in_on_a_secret_that_s_dramatic_irony_christopher_warner  4.57M  9.38G  1.12M   128K 20130129 2 min 49 s
                                   what_is_obesity_mia_nacamulli  8.40M  9.38G   765K   127K 20160630 5 min 10 s
       what_in_the_world_is_topological_quantum_matter_fan_zhang  7.40M  9.39G   507K   127K 20171023 5 min 2 s
                     hacking_bacteria_to_fight_cancer_tal_danino  11.5M  9.40G   251K   125K 20191210 5 min 10 s
             why_should_you_read_toni_morrisons_beloved_yen_pham  20.9M  9.42G   125K   125K 20210105 5 min 6 s
    the_life_legacy_assassination_of_an_african_revolutionary_li  18.6M  9.44G   249K   124K 20200203 5 min 31 s
            rosalind_franklin_dna_s_unsung_hero_claudio_l_guerra  7.82M  9.45G   742K   124K 20160711 4 min 9 s
       inside_okcupid_the_math_of_online_dating_christian_rudder  16.5M  9.47G  1.09M   124K 20130213 7 min 30 s
                    what_happened_to_trial_by_jury_suja_a_thomas  7.02M  9.47G   617K   123K 20170302 4 min 11 s
                        why_don_t_oil_and_water_mix_john_pollard  14.7M  9.49G   982K   123K 20131010 5 min 2 s
                         can_steroids_save_your_life_anees_bahji  18.8M  9.50G   245K   123K 20200617 5 min 31 s
                                why_do_we_pass_gas_purna_kashyap  13.2M  9.52G   855K   122K 20140908 4 min 57 s
                         what_are_mini_brains_madeline_lancaster  6.85M  9.52G   488K   122K 20180116 4 min 44 s
    nasas_first_software_engineer_margaret_hamilton_matt_porter_  10.1M  9.53G   244K   122K 20200305 5 min 9 s
        is_there_a_limit_to_technological_progress_clement_vidal  8.32M  9.54G   609K   122K 20161219 4 min 46 s
               the_neuroscience_of_imagination_andrey_vyshedskiy  17.5M  9.56G   609K   122K 20161212 4 min 48 s
                            on_reading_the_koran_lesley_hazleton  13.7M  9.57G  1.07M   121K 20130825 9 min 33 s
              how_do_we_know_what_color_dinosaurs_were_len_bloch  7.76M  9.58G   728K   121K 20160104 4 min 23 s
           are_naked_mole_rats_the_strangest_mammals_thomas_park  9.09M  9.59G   485K   121K 20180529 4 min 46 s
                          the_infinite_life_of_pi_reynaldo_lopes  6.66M  9.60G  1.06M   120K 20130710 3 min 44 s
                     is_light_a_particle_or_a_wave_colm_kelleher  10.4M  9.61G  1.06M   120K 20130117 4 min 23 s
          how_does_the_thyroid_manage_your_metabolism_emma_bryce  7.16M  9.61G   837K   120K 20150302 3 min 36 s
                           how_transistors_work_gokul_j_krishnan  6.48M  9.62G   712K   119K 20160606 4 min 53 s
                         is_graffiti_art_or_vandalism_kelly_wall  15.2M  9.63G   591K   118K 20160908 4 min 31 s
    could_human_civilization_spread_across_the_whole_galaxy_roey  8.02M  9.64G   591K   118K 20160929 4 min 33 s
                       sugar_hiding_in_plain_sight_robert_lustig  9.30M  9.65G   944K   118K 20140331 4 min 3 s
                         the_art_of_the_metaphor_jane_hirshfield  9.29M  9.66G  1.04M   118K 20120924 5 min 38 s
        the_simple_story_of_photosynthesis_and_food_amanda_ooten  8.50M  9.67G  1.03M   117K 20130305 4 min 0 s
                             the_factory_think_like_a_coder_ep_9  41.0M  9.71G   234K   117K 20200623 10 min 0 s
                        try_something_new_for_30_days_matt_cutts  5.88M  9.71G  1.03M   117K 20130405 3 min 27 s
    particles_and_waves_the_central_mystery_of_quantum_mechanics  7.11M  9.72G   814K   116K 20140915 4 min 51 s
                            how_did_english_evolve_kate_gardoqui  15.5M  9.74G  1.02M   116K 20121127 5 min 4 s
                  how_did_clouds_get_their_names_richard_hamblyn  9.26M  9.75G   692K   115K 20151124 5 min 6 s
    how_the_band_aid_was_invented_moments_of_vision_3_jessica_or  3.05M  9.75G   576K   115K 20160912 1 min 44 s
                  biodiesel_the_afterlife_of_oil_natascia_radice  7.48M  9.76G   921K   115K 20140123 4 min 14 s
                the_city_of_walls_constantinople_lars_brownworth  6.94M  9.76G  1.01M   115K 20121018 4 min 16 s
                               music_as_a_language_victor_wooten  15.5M  9.78G  1.11M   114K 20120813 4 min 59 s
            the_upside_of_isolated_civilizations_jason_shipinski  8.37M  9.79G  1.00M   113K 20130328 4 min 7 s
                                 how_to_grow_a_glacier_m_jackson  8.76M  9.79G   340K   113K 20190404 5 min 19 s
     why_we_love_repetition_in_music_elizabeth_hellmuth_margulis  8.06M  9.80G   904K   113K 20140902 4 min 31 s
                       does_stress_cause_pimples_claudia_aguirre  6.20M  9.81G  0.99M   113K 20121115 3 min 54 s
    how_do_us_supreme_court_justices_get_appointed_peter_paccone  8.46M  9.82G   564K   113K 20161117 4 min 25 s
      how_one_scientist_took_on_the_chemical_industry_mark_lytle  12.0M  9.83G   225K   112K 20200317 5 min 22 s
                          why_are_fish_fish_shaped_lauren_sallan  11.9M  9.84G   449K   112K 20180417 4 min 56 s
    did_shakespeare_write_his_plays_natalya_st_clair_and_aaron_w  6.64M  9.85G   785K   112K 20150224 4 min 6 s
                       how_do_birds_learn_to_sing_partha_p_mitra  11.3M  9.86G   448K   112K 20180220 5 min 38 s
    the_electrifying_speeches_of_sojourner_truth_daina_ramey_ber  9.68M  9.87G   223K   111K 20200428 4 min 39 s
                    how_to_sequence_the_human_genome_mark_j_kiel  12.7M  9.88G   887K   111K 20131209 5 min 4 s
                       how_to_3d_print_human_tissue_taneka_jones  7.60M  9.89G   221K   111K 20191017 5 min 11 s
    the_mighty_mathematics_of_the_lever_andy_peterson_and_zack_p  7.27M  9.89G   775K   111K 20141118 4 min 45 s
                       surviving_a_nuclear_attack_irwin_redlener  44.7M  9.94G   990K   110K 20130125 25 min 21 s
      calculating_the_odds_of_intelligent_alien_life_jill_tarter  20.3M  9.96G  1.07M   110K 20120702 7 min 27 s
    explore_cave_paintings_in_this_360deg_animated_cave_iseult_g  4.55M  9.96G   438K   110K 20171012 3 min 0 s
      how_the_bra_was_invented_moments_of_vision_1_jessica_oreck  3.36M  9.96G   654K   109K 20160712 1 min 42 s
    can_you_spot_the_problem_with_these_headlines_level_1_jeff_l  9.34M  9.97G   327K   109K 20190521 5 min 0 s
                   what_creates_a_total_solar_eclipse_andy_cohen  5.99M  9.98G   979K   109K 20130722 3 min 45 s
                    deep_ocean_mysteries_and_wonders_david_gallo  22.5M  10.0G  1.05M   108K 20120312 8 min 27 s
       how_taking_a_bath_led_to_archimedes_principle_mark_salata  8.23M  10.0G  1.05M   107K 20120906 3 min 0 s
    the_high_stakes_race_to_make_quantum_computers_work_chiara_d  9.41M  10.0G   321K   107K 20190813 5 min 15 s
                                      how_heavy_is_air_dan_quinn  9.90M  10.0G   853K   107K 20140707 3 min 18 s
               the_beauty_of_data_visualization_david_mccandless  31.2M  10.1G   958K   106K 20121123 18 min 17 s
                            how_do_focus_groups_work_hector_lanz  9.05M  10.1G   531K   106K 20170410 4 min 46 s
          beware_of_nominalizations_aka_zombie_nouns_helen_sword  11.3M  10.1G   952K   106K 20121031 5 min 4 s
                            how_to_spot_a_fad_diet_mia_nacamulli  7.10M  10.1G   634K   106K 20160411 4 min 33 s
    what_makes_neon_signs_glow_a_360deg_animation_michael_lipman  9.23M  10.1G   211K   106K 20190919 4 min 51 s
         the_beginning_of_the_universe_for_beginners_tom_whyntie  5.87M  10.1G   945K   105K 20130409 3 min 41 s
                         the_end_of_history_illusion_bence_nanay  5.94M  10.1G   314K   105K 20180927 4 min 22 s
                    newton_s_3_laws_with_a_bicycle_joshua_manley  10.6M  10.1G   939K   104K 20120919 3 min 32 s
      how_does_your_smartphone_know_your_location_wilton_l_virgo  9.32M  10.1G   730K   104K 20150129 5 min 2 s
           what_is_the_biggest_single_celled_organism_murry_gans  6.89M  10.1G   624K   104K 20160818 4 min 6 s
                         five_fingers_of_evolution_paul_andersen  10.5M  10.1G  1.01M   104K 20120507 5 min 23 s
              can_wildlife_adapt_to_climate_change_erin_eastwood  10.7M  10.2G   620K   103K 20160303 4 min 46 s
                       how_art_can_help_you_analyze_amy_e_herman  7.45M  10.2G   815K   102K 20131004 4 min 49 s
      can_you_find_the_next_number_in_this_sequence_alex_gendler  6.83M  10.2G   506K   101K 20170720 4 min 0 s
    how_to_squeeze_electricity_out_of_crystals_ashwini_bharathul  13.1M  10.2G   506K   101K 20170620 4 min 55 s
                        how_do_self_driving_cars_see_sajan_saini  9.03M  10.2G   303K   101K 20190513 5 min 24 s
       why_is_the_us_constitution_so_hard_to_amend_peter_paccone  8.32M  10.2G   603K   101K 20160516 4 min 17 s
                      a_reality_check_on_renewables_david_mackay  33.4M  10.2G   901K   100K 20130626 18 min 34 s
             a_brief_history_of_video_games_part_i_safwat_saleem  6.97M  10.2G   900K  99.9K 20130813 4 min 45 s
    cell_membranes_are_way_more_complicated_than_you_think_nazzy  14.6M  10.3G   500K  99.9K 20170821 5 min 20 s
                 eli_the_eel_a_mysterious_migration_james_prosek  6.61M  10.3G   798K  99.8K 20140210 4 min 38 s
                                                         why_sex  12.5M  10.3G   997K  99.7K 20120501 4 min 52 s
    how_do_germs_spread_and_why_do_they_make_us_sick_yannay_khai  6.55M  10.3G   698K  99.7K 20141021 5 min 6 s
    the_exceptional_life_of_benjamin_banneker_rose_margaret_eken  6.56M  10.3G   498K  99.6K 20170216 3 min 36 s
    what_did_democracy_really_mean_in_athens_melissa_schwartzber  19.0M  10.3G   697K  99.5K 20150324 4 min 51 s
    how_exposing_anonymous_companies_could_cut_down_on_crime_glo  10.1M  10.3G   592K  98.6K 20151201 4 min 6 s
                        how_to_fossilize_yourself_phoebe_a_cohen  8.15M  10.3G   785K  98.1K 20140106 5 min 13 s
                               the_physics_of_surfing_nick_pizzo  11.1M  10.3G   294K  97.9K 20190311 4 min 40 s
                             how_to_choose_your_news_damon_brown  6.41M  10.3G   780K  97.5K 20140605 4 min 48 s
    buffalo_buffalo_buffalo_one_word_sentences_and_how_they_work  5.00M  10.3G   682K  97.4K 20150817 3 min 27 s
                                  how_we_see_color_colm_kelleher  10.7M  10.4G   875K  97.2K 20130108 3 min 43 s
                           when_to_use_apostrophes_laura_mcclure  5.63M  10.4G   680K  97.1K 20150727 3 min 13 s
                         would_you_live_on_the_moon_alex_gendler  9.05M  10.4G   388K  97.0K 20180605 4 min 51 s
    the_incredible_collaboration_behind_the_international_space_  9.79M  10.4G   580K  96.7K 20150929 4 min 57 s
                           what_is_the_world_wide_web_twila_camp  6.02M  10.4G   773K  96.7K 20140512 3 min 54 s
                                              the_opposites_game  13.1M  10.4G   289K  96.3K 20190603 4 min 40 s
           a_guide_to_the_energy_of_the_earth_joshua_m_sneideman  9.00M  10.4G   769K  96.2K 20140630 4 min 43 s
      a_clever_way_to_estimate_enormous_numbers_michael_mitchell  10.1M  10.4G   865K  96.2K 20120912 4 min 14 s
               why_the_insect_brain_is_so_incredible_anna_stockl  12.0M  10.4G   576K  96.0K 20160414 4 min 22 s
           climate_change_earth_s_giant_game_of_tetris_joss_fong  4.57M  10.4G   767K  95.9K 20140422 2 min 48 s
    the_fundamentals_of_space_time_part_2_andrew_pontzen_and_tom  8.17M  10.4G   763K  95.4K 20140501 4 min 49 s
         how_do_we_separate_the_seemingly_inseparable_iddo_magen  6.67M  10.4G   569K  94.9K 20160509 4 min 23 s
            how_to_master_your_sense_of_smell_alexandra_horowitz  9.71M  10.5G   474K  94.8K 20170109 4 min 33 s
    are_spotty_fruits_and_vegetables_safe_to_eat_elizabeth_braue  7.13M  10.5G   565K  94.2K 20160822 4 min 8 s
                      what_causes_economic_bubbles_prateek_singh  9.45M  10.5G   658K  93.9K 20150504 4 min 16 s
                           questions_no_one_knows_the_answers_to  4.20M  10.5G   937K  93.7K 20120311 2 min 10 s
                  all_the_world_s_a_stage_by_william_shakespeare  5.65M  10.5G   281K  93.6K 20190202 2 min 34 s
             the_power_of_creative_constraints_brandon_rodriguez  7.08M  10.5G   463K  92.7K 20170613 5 min 9 s
                   the_terrors_of_sleep_paralysis_ami_angelowicz  10.7M  10.5G   828K  92.0K 20130725 4 min 48 s
              what_is_leukemia_danilo_allegra_and_dania_puggioni  7.89M  10.5G   643K  91.8K 20150430 4 min 32 s
                  kabuki_the_people_s_dramatic_art_amanda_mattes  7.30M  10.5G   733K  91.6K 20130930 4 min 15 s
                            what_does_the_pancreas_do_emma_bryce  7.01M  10.5G   641K  91.5K 20150219 3 min 20 s
               the_story_behind_the_boston_tea_party_ben_labaree  10.0M  10.5G   821K  91.2K 20130318 3 min 47 s
                          how_do_you_know_whom_to_trust_ram_neta  10.2M  10.5G   818K  90.9K 20130430 4 min 34 s
                                       first_kiss_by_tim_seibles  5.95M  10.5G   272K  90.6K 20190401 2 min 37 s
    the_uncertain_location_of_electrons_george_zaidan_and_charle  5.84M  10.5G   720K  90.0K 20131014 3 min 46 s
    romance_and_revolution_the_poetry_of_pablo_neruda_ilan_stava  15.3M  10.6G   270K  90.0K 20190723 4 min 49 s
    should_we_be_looking_for_life_elsewhere_in_the_universe_aoma  8.84M  10.6G   537K  89.5K 20160725 4 min 35 s
          how_do_schools_of_fish_swim_in_harmony_nathan_s_jacobs  13.9M  10.6G   535K  89.2K 20160331 6 min 6 s
            solving_the_puzzle_of_the_periodic_table_eric_rosado  9.56M  10.6G   798K  88.7K 20121212 4 min 18 s
                        the_evolution_of_the_book_julie_dreyfuss  12.1M  10.6G   531K  88.5K 20160613 4 min 17 s
            infinity_according_to_jorge_luis_borges_ilan_stavans  7.98M  10.6G   265K  88.3K 20190711 4 min 42 s
                            where_do_genes_come_from_carl_zimmer  8.08M  10.6G   615K  87.9K 20140922 4 min 23 s
           why_should_you_read_flannery_oconnor_iseult_gillespie  7.06M  10.6G   263K  87.7K 20190129 4 min 11 s
                the_pleasure_of_poetic_pattern_david_silverstein  8.93M  10.6G   525K  87.6K 20160602 4 min 46 s
                     why_are_blue_whales_so_enormous_asha_de_vos  11.7M  10.6G   787K  87.4K 20130225 5 min 20 s
                 how_north_america_got_its_shape_peter_j_haproff  8.40M  10.7G   524K  87.4K 20160705 4 min 57 s
                           the_higgs_field_explained_don_lincoln  11.4M  10.7G   786K  87.3K 20130827 3 min 18 s
              sunlight_is_way_older_than_you_think_sten_odenwald  7.77M  10.7G   610K  87.1K 20150512 4 min 36 s
    diagnosing_a_zombie_brain_and_body_part_one_tim_verstynen_br  6.56M  10.7G   783K  87.0K 20121022 3 min 46 s
          got_seeds_just_add_bleach_acid_and_sandpaper_mary_koga  4.99M  10.7G   779K  86.5K 20130716 3 min 35 s
    the_last_chief_of_the_comanches_and_the_fall_of_an_empire_du  22.7M  10.7G   172K  86.0K 20200702 6 min 24 s
    what_s_hidden_among_the_tallest_trees_on_earth_wendell_oshir  11.6M  10.7G   685K  85.7K 20140805 4 min 46 s
      how_braille_was_invented_moments_of_vision_9_jessica_oreck  2.98M  10.7G   428K  85.6K 20170307 1 min 49 s
    if_matter_falls_down_does_antimatter_fall_up_chloe_malbrunot  7.29M  10.7G   597K  85.2K 20140929 2 min 54 s
    the_infamous_and_ingenious_ho_chi_minh_trail_cameron_paterso  13.7M  10.7G   762K  84.7K 20130314 3 min 54 s
      ode_to_the_only_black_kid_in_the_class_poem_by_clint_smith  1.76M  10.7G   169K  84.6K 20190909 1 min 8 s
    how_blue_jeans_were_invented_moments_of_vision_10_jessica_or  3.82M  10.7G   422K  84.3K 20170406 1 min 56 s
    why_should_you_read_the_joy_luck_club_by_amy_tan_sheila_mari  6.08M  10.8G   168K  84.2K 20191216 3 min 52 s
                  could_your_brain_repair_itself_ralitsa_petrova  6.33M  10.8G   587K  83.8K 20150427 3 min 59 s
    the_strengths_and_weaknesses_of_acids_and_bases_george_zaida  6.84M  10.8G   668K  83.5K 20131024 3 min 47 s
    underwater_farms_vs_climate_change_ayana_elizabeth_johnson_a  12.2M  10.8G   250K  83.5K 20190613 4 min 30 s
                 a_brief_history_of_plural_word_s_john_mcwhorter  8.01M  10.8G   749K  83.2K 20130722 4 min 26 s
    why_wasnt_the_bill_of_rights_originally_in_the_us_constituti  9.62M  10.8G   499K  83.1K 20160614 4 min 32 s
                                logarithms_explained_steve_kelly  5.67M  10.8G   826K  82.6K 20120820 3 min 33 s
                 your_body_language_shapes_who_you_are_amy_cuddy  54.3M  10.9G   740K  82.2K 20130608 21 min 2 s
    the_case_of_the_missing_fractals_alex_rosenthal_and_george_z  21.3M  10.9G   655K  81.8K 20140429 4 min 52 s
                   why_are_manhole_covers_round_marc_chamberland  6.85M  10.9G   572K  81.7K 20150413 3 min 34 s
    the_historic_womens_suffrage_march_on_washington_michelle_me  10.7M  10.9G   244K  81.4K 20190304 4 min 54 s
                                           the_cockroach_beatbox  15.2M  10.9G   810K  81.0K 20120311 6 min 15 s
                how_we_think_complex_cells_evolved_adam_jacobson  9.84M  10.9G   567K  81.0K 20150217 5 min 41 s
        how_do_brain_scans_work_john_borghi_and_elizabeth_waters  9.10M  10.9G   323K  80.8K 20180426 4 min 59 s
           is_dna_the_future_of_data_storage_leo_bear_mcguinness  18.0M  10.9G   318K  79.6K 20171009 5 min 29 s
    how_the_popsicle_was_invented_moments_of_vision_11_jessica_o  3.61M  10.9G   396K  79.2K 20170502 1 min 50 s
        the_most_lightning_struck_place_on_earth_graeme_anderson  7.34M  11.0G   474K  79.1K 20160128 3 min 40 s
    gerrymandering_how_drawing_jagged_lines_can_impact_an_electi  7.17M  11.0G   707K  78.6K 20121025 3 min 52 s
                  the_case_of_the_vanishing_honeybees_emma_bryce  11.5M  11.0G   624K  78.0K 20140318 3 min 46 s
                                     what_is_color_colm_kelleher  7.20M  11.0G   700K  77.8K 20121218 3 min 9 s
                       the_second_coming_by_william_butler_yeats  2.32M  11.0G   232K  77.5K 20190202 1 min 56 s
                 how_atoms_bond_george_zaidan_and_charles_morton  5.04M  11.0G   617K  77.1K 20131015 3 min 33 s
    how_the_bendy_straw_was_invented_moments_of_vision_12_jessic  3.05M  11.0G   384K  76.7K 20170601 1 min 39 s
                         how_do_we_study_the_stars_yuan_sen_ting  9.96M  11.0G   532K  76.0K 20141007 4 min 45 s
                               the_pangaea_pop_up_michael_molina  6.78M  11.0G   607K  75.8K 20140203 4 min 25 s
                 why_is_it_so_hard_to_cure_als_fernando_g_vieira  10.4M  11.0G   303K  75.7K 20180531 5 min 21 s
            light_waves_visible_and_invisible_lucianne_walkowicz  12.0M  11.0G   600K  75.0K 20130919 5 min 57 s
            radioactivity_expect_the_unexpected_steve_weatherall  9.30M  11.0G   675K  75.0K 20121210 4 min 15 s
                                       accents_by_denice_frohman  9.39M  11.0G   225K  74.9K 20190502 2 min 31 s
    the_history_of_african_american_social_dance_camille_a_brown  17.2M  11.1G   375K  74.9K 20160927 4 min 52 s
    from_the_top_of_the_food_chain_down_rewilding_our_world_geor  10.1M  11.1G   599K  74.8K 20140313 5 min 27 s
                how_to_build_a_dark_matter_detector_jenna_saffin  11.5M  11.1G   299K  74.7K 20180501 4 min 32 s
              could_a_breathalyzer_detect_cancer_julian_burschka  7.64M  11.1G   148K  74.2K 20200106 4 min 39 s
                          the_moon_illusion_andrew_vanden_heuvel  8.19M  11.1G   667K  74.1K 20130903 4 min 8 s
                          reasons_for_the_seasons_rebecca_kaplan  8.67M  11.1G   662K  73.5K 20130523 5 min 20 s
                                rethinking_thinking_trevor_maber  13.7M  11.1G   661K  73.4K 20121015 5 min 32 s
                            for_estefani_poem_by_aracelis_girmay  13.8M  11.1G   146K  73.2K 20191003 3 min 27 s
                       a_brief_history_of_religion_in_art_ted_ed  14.8M  11.2G   585K  73.1K 20140616 4 min 37 s
           the_dangerous_race_for_the_south_pole_elizabeth_leane  7.29M  11.2G   218K  72.6K 20181210 4 min 47 s
                        why_it_pays_to_work_hard_richard_st_john  15.8M  11.2G   652K  72.4K 20130405 6 min 22 s
                     solid_liquid_gas_and_plasma_michael_murillo  15.6M  11.2G   505K  72.1K 20150728 3 min 51 s
    how_a_few_scientists_transformed_the_way_we_think_about_dise  9.78M  11.2G   432K  72.0K 20151020 4 min 38 s
    looks_aren_t_everything_believe_me_i_m_a_model_cameron_russe  19.4M  11.2G   643K  71.5K 20130526 9 min 37 s
    how_polarity_makes_water_behave_strangely_christina_kleinber  7.16M  11.2G   642K  71.3K 20130204 3 min 51 s
    why_do_hospitals_have_particle_accelerators_pedro_brugarolas  6.70M  11.2G   214K  71.2K 20190319 4 min 46 s
                                 life_of_an_astronaut_jerry_carr  8.86M  11.2G   641K  71.2K 20130130 4 min 51 s
    how_super_glue_was_invented_moments_of_vision_8_jessica_orec  3.37M  11.2G   355K  71.0K 20170206 1 min 53 s
                do_we_really_need_pesticides_fernan_perez_galvez  10.1M  11.3G   354K  70.7K 20161114 5 min 17 s
    why_should_you_read_sci_fi_superstar_octavia_e_butler_ayana_  9.27M  11.3G   212K  70.7K 20190225 4 min 14 s
    what_aristotle_and_joshua_bell_can_teach_us_about_persuasion  9.96M  11.3G   635K  70.6K 20130114 4 min 39 s
    why_aren_t_we_only_using_solar_power_alexandros_george_chara  10.8M  11.3G   564K  70.5K 20140619 4 min 42 s
                           to_make_use_of_water_by_safia_elhillo  4.22M  11.3G   212K  70.5K 20190202 2 min 4 s
        how_coffee_got_quicker_moments_of_vision_2_jessica_oreck  4.18M  11.3G   421K  70.1K 20160808 1 min 47 s
                 what_triggers_a_chemical_reaction_kareem_jarrah  6.09M  11.3G   490K  70.0K 20150120 3 min 45 s
                 the_battle_of_the_greek_tragedies_melanie_sirof  13.7M  11.3G   489K  69.9K 20150601 5 min 6 s
                           the_science_of_symmetry_colm_kelleher  15.5M  11.3G   556K  69.5K 20140513 5 min 8 s
    dead_stuff_the_secret_ingredient_in_our_food_chain_john_c_mo  8.00M  11.3G   556K  69.5K 20140320 3 min 50 s
    how_the_stethoscope_was_invented_moments_of_vision_7_jessica  3.79M  11.3G   347K  69.5K 20161229 1 min 48 s
                    pixar_the_math_behind_the_movies_tony_derose  15.8M  11.4G   544K  68.0K 20140325 7 min 33 s
                    why_do_animals_form_swarms_maria_r_d_orsogna  18.5M  11.4G   272K  68.0K 20171218 4 min 5 s
        what_can_you_learn_from_ancient_skeletons_farnaz_khatibi  8.48M  11.4G   339K  67.9K 20170615 4 min 7 s
     the_dust_bunnies_that_built_our_planet_lorin_swint_matthews  12.9M  11.4G   203K  67.7K 20190905 5 min 30 s
     an_unsung_hero_of_the_civil_rights_movement_christina_greer  9.65M  11.4G   202K  67.3K 20190221 4 min 29 s
                            why_is_there_a_b_in_doubt_gina_cooke  4.63M  11.4G   606K  67.3K 20121217 3 min 27 s
                could_we_survive_prolonged_space_travel_lisa_nip  12.7M  11.4G   335K  67.0K 20161004 4 min 55 s
    how_the_rubber_glove_was_invented_moments_of_vision_4_jessic  3.30M  11.4G   335K  67.0K 20161011 1 min 34 s
          how_misused_modifiers_can_hurt_your_writing_emma_bryce  5.25M  11.4G   401K  66.8K 20150908 3 min 20 s
                             how_did_feathers_evolve_carl_zimmer  6.93M  11.4G   598K  66.5K 20130502 3 min 26 s
               the_race_to_sequence_the_human_genome_tien_nguyen  10.8M  11.4G   393K  65.6K 20151012 4 min 59 s
               which_sunscreen_should_you_choose_mary_poffenroth  8.04M  11.5G   393K  65.4K 20160801 4 min 39 s
                the_first_asteroid_ever_discovered_carrie_nugent  7.50M  11.5G   262K  65.4K 20171016 5 min 5 s
                                 everyday_leadership_drew_dudley  12.9M  11.5G   587K  65.2K 20130815 6 min 14 s
    the_fundamentals_of_space_time_part_3_andrew_pontzen_and_tom  5.63M  11.5G   521K  65.2K 20140522 3 min 26 s
    feedback_loops_how_nature_gets_its_rhythms_anje_margriet_neu  14.5M  11.5G   517K  64.7K 20140825 5 min 10 s
           how_much_does_a_video_weigh_michael_stevens_of_vsauce  15.4M  11.5G   581K  64.5K 20130424 7 min 20 s
                         the_science_of_snowflakes_marusa_bradac  7.73M  11.5G   386K  64.4K 20151222 4 min 29 s
      is_there_a_reproducibility_crisis_in_science_matt_anticole  9.31M  11.5G   322K  64.4K 20161205 4 min 46 s
                               the_nutritionist_by_andrea_gibson  8.67M  11.5G   193K  64.3K 20190202 4 min 44 s
    what_is_chemical_equilibrium_george_zaidan_and_charles_morto  8.02M  11.5G   576K  64.0K 20130723 3 min 24 s
           animation_basics_the_art_of_timing_and_spacing_ted_ed  12.2M  11.6G   509K  63.7K 20140128 6 min 42 s
         are_there_universal_expressions_of_emotion_sophie_zadeh  8.79M  11.6G   255K  63.6K 20180703 4 min 51 s
                            how_brass_instruments_work_al_cannon  12.8M  11.6G   444K  63.4K 20150407 4 min 11 s
                          when_to_use_me_myself_and_i_emma_bryce  5.59M  11.6G   379K  63.2K 20150914 2 min 56 s
    the_weird_and_wonderful_metamorphosis_of_the_butterfly_franz  10.4M  11.6G   252K  62.9K 20180301 5 min 2 s
                       are_shakespeare_s_plays_encoded_within_pi  12.2M  11.6G   629K  62.9K 20120313 3 min 45 s
    is_there_a_center_of_the_universe_marjee_chmiel_and_trevor_o  9.33M  11.6G   563K  62.5K 20130625 4 min 13 s
    corruption_wealth_and_beauty_the_history_of_the_venetian_gon  11.4M  11.6G   499K  62.3K 20140904 4 min 50 s
                             overcoming_obstacles_steven_claunch  11.5M  11.6G   559K  62.1K 20130821 4 min 22 s
       is_there_a_difference_between_art_and_craft_laura_morelli  12.2M  11.6G   490K  61.2K 20140306 5 min 30 s
                  what_is_abstract_expressionism_sarah_rosenthal  15.2M  11.7G   365K  60.8K 20160428 4 min 49 s
            why_do_people_have_seasonal_allergies_eleanor_nelsen  15.2M  11.7G   365K  60.8K 20160526 5 min 1 s
       harvey_milk_s_radical_vision_of_equality_lillian_faderman  9.26M  11.7G   182K  60.7K 20190312 5 min 23 s
               music_and_creativity_in_ancient_greece_tim_hansen  9.58M  11.7G   484K  60.5K 20131203 4 min 45 s
     how_computers_translate_human_language_ioannis_papachimonas  11.6M  11.7G   360K  60.1K 20151026 4 min 44 s
              why_the_shape_of_your_screen_matters_brian_gervase  5.26M  11.7G   539K  59.9K 20120913 3 min 32 s
                   become_a_slam_poet_in_five_steps_gayle_danley  11.5M  11.7G   535K  59.4K 20130327 3 min 31 s
                        why_tragedies_are_alluring_david_e_rivas  9.11M  11.7G   413K  59.0K 20150709 4 min 25 s
       the_chemical_reaction_that_feeds_the_world_daniel_d_dulek  10.7M  11.7G   472K  59.0K 20131118 5 min 18 s
    the_effects_of_underwater_pressure_on_the_body_neosha_s_kash  6.97M  11.7G   411K  58.8K 20150402 4 min 2 s
    the_fight_for_the_right_to_vote_in_the_united_states_nicki_b  6.60M  11.8G   470K  58.8K 20131105 4 min 30 s
                           how_to_create_cleaner_coal_emma_bryce  9.86M  11.8G   411K  58.7K 20141209 5 min 53 s
                          what_happens_if_you_guess_leigh_nataro  7.57M  11.8G   583K  58.3K 20120831 5 min 27 s
    the_many_meanings_of_michelangelo_s_statue_of_david_james_ea  5.54M  11.8G   459K  57.4K 20140715 3 min 18 s
                      pizza_physics_new_york_style_colm_kelleher  7.80M  11.8G   516K  57.4K 20121206 3 min 57 s
    the_controversial_origins_of_the_encyclopedia_addison_anders  12.6M  11.8G   342K  57.0K 20160218 5 min 20 s
    how_quantum_mechanics_explains_global_warming_lieven_scheire  17.5M  11.8G   455K  56.9K 20140717 5 min 0 s
                  a_simple_way_to_tell_insects_apart_anika_hazra  6.46M  11.8G   226K  56.6K 20180403 4 min 6 s
    how_smudge_proof_lipstick_was_invented_moments_of_vision_6_j  4.12M  11.8G   283K  56.5K 20161206 2 min 5 s
                             can_robots_be_creative_gil_weinberg  8.49M  11.8G   393K  56.2K 20150319 5 min 26 s
                                    new_colossus_by_emma_lazarus  1.98M  11.8G   168K  56.1K 20190702 1 min 24 s
                   what_is_the_universe_made_of_dennis_wildfogel  11.3M  11.8G   445K  55.7K 20140225 4 min 4 s
                              why_do_we_have_museums_j_v_maranto  11.7M  11.9G   387K  55.2K 20150205 5 min 43 s
                              how_to_live_to_be_100_dan_buettner  32.8M  11.9G   492K  54.7K 20130417 19 min 39 s
    the_suns_surprising_movement_across_the_sky_gordon_williamso  7.46M  11.9G   328K  54.7K 20151221 4 min 22 s
        the_popularity_plight_and_poop_of_penguins_dyan_denapoli  12.8M  11.9G   437K  54.6K 20131217 5 min 23 s
                can_machines_read_your_emotions_kostas_karpouzis  7.50M  11.9G   273K  54.6K 20161129 4 min 38 s
    not_all_scientific_studies_are_created_equal_david_h_schwart  6.23M  11.9G   436K  54.5K 20140428 4 min 26 s
                  the_power_of_a_great_introduction_carolyn_mohr  10.4M  11.9G   490K  54.5K 20120927 4 min 42 s
                       why_neutrinos_matter_silvia_bravo_gallart  6.62M  11.9G   374K  53.4K 20150428 4 min 40 s
             the_making_of_the_american_constitution_judy_walton  7.78M  11.9G   480K  53.3K 20121023 3 min 57 s
                            the_power_of_passion_richard_st_john  15.5M  12.0G   479K  53.2K 20130405 6 min 54 s
                                walking_on_eggs_sick_science_069  2.07M  12.0G   532K  53.2K 20120104 1 min 5 s
    how_to_visualize_one_part_per_million_kim_preshoff_the_ted_e  5.68M  12.0G   318K  53.1K 20160815 2 min 27 s
    how_inventions_change_history_for_better_and_for_worse_kenne  12.5M  12.0G   477K  53.0K 20121017 5 min 14 s
                     how_fiction_can_change_reality_jessica_wise  8.99M  12.0G   529K  52.9K 20120823 4 min 29 s
          will_future_spacecraft_fit_in_our_pockets_dhonam_pemba  8.31M  12.0G   370K  52.8K 20150528 4 min 36 s
                                              big_data_tim_smith  12.9M  12.0G   474K  52.7K 20130503 6 min 7 s
    learning_from_smallpox_how_to_eradicate_a_disease_julie_garo  9.37M  12.0G   366K  52.3K 20150310 5 min 44 s
                    can_plants_talk_to_each_other_richard_karban  14.5M  12.0G   312K  52.0K 20160502 4 min 38 s
      four_ways_to_understand_the_earth_s_age_joshua_m_sneideman  10.1M  12.0G   466K  51.8K 20130829 3 min 44 s
                             what_is_a_gift_economy_alex_gendler  7.23M  12.0G   356K  50.8K 20141223 4 min 5 s
                  is_our_universe_the_only_universe_brian_greene  44.1M  12.1G   455K  50.6K 20130419 21 min 47 s
                ted_invites_the_class_of_2020_to_give_a_ted_talk  3.54M  12.1G   100K  50.0K 20200511 1 min 43 s
    how_i_responded_to_sexism_in_gaming_with_empathy_lilian_chen  10.8M  12.1G   347K  49.6K 20150526 6 min 59 s
         what_s_the_definition_of_comedy_banana_addison_anderson  7.20M  12.1G   439K  48.8K 20130730 4 min 50 s
          the_coelacanth_a_living_fossil_of_a_fish_erin_eastwood  10.7M  12.1G   386K  48.2K 20140729 4 min 16 s
    the_punishable_perils_of_plagiarism_melissa_huseman_d_annunz  8.44M  12.1G   429K  47.7K 20130614 3 min 47 s
                            the_power_of_simple_words_terin_izil  3.90M  12.1G   476K  47.6K 20120311 2 min 1 s
    the_microbial_jungles_all_over_the_place_and_you_scott_chimi  10.3M  12.1G   285K  47.5K 20160517 5 min 10 s
                               the_true_story_of_true_gina_cooke  16.2M  12.2G   380K  47.5K 20131216 4 min 27 s
                        the_chemistry_of_cold_packs_john_pollard  8.04M  12.2G   331K  47.2K 20140911 4 min 31 s
                                             electric_vocabulary  12.9M  12.2G   467K  46.7K 20120716 6 min 56 s
                                  dna_the_book_of_you_joe_hanson  13.5M  12.2G   419K  46.6K 20121126 4 min 28 s
                tycho_brahe_the_scandalous_astronomer_dan_wenkel  11.4M  12.2G   371K  46.3K 20140612 4 min 7 s
              what_s_below_the_tip_of_the_iceberg_camille_seaman  8.67M  12.2G   416K  46.3K 20130724 4 min 51 s
                              birth_of_a_nickname_john_mcwhorter  7.97M  12.2G   369K  46.1K 20130924 4 min 56 s
         how_to_organize_add_and_multiply_matrices_bill_shillito  7.01M  12.2G   414K  46.0K 20130304 4 min 40 s
    the_historical_audacity_of_the_louisiana_purchase_judy_walto  11.2M  12.2G   414K  46.0K 20130207 3 min 38 s
    the_operating_system_of_life_george_zaidan_and_charles_morto  7.67M  12.2G   366K  45.8K 20131111 4 min 0 s
                        introducing_ted_ed_lessons_worth_sharing  6.73M  12.3G   458K  45.8K 20120312 2 min 11 s
                         the_importance_of_focus_richard_st_john  13.8M  12.3G   411K  45.7K 20130306 5 min 54 s
                how_people_rationalize_fraud_kelly_richmond_pope  13.4M  12.3G   317K  45.3K 20150608 4 min 34 s
    gyotaku_the_ancient_japanese_art_of_printing_fish_k_erica_do  9.13M  12.3G   407K  45.2K 20130530 3 min 37 s
                                what_is_an_aurora_michael_molina  12.8M  12.3G   403K  44.7K 20130703 4 min 9 s
            how_ancient_art_influenced_modern_art_felipe_galindo  13.9M  12.3G   268K  44.7K 20160225 4 min 50 s
         the_contributions_of_female_explorers_courtney_stephens  9.09M  12.3G   400K  44.4K 20130612 4 min 25 s
              the_physics_of_playing_guitar_oscar_fernando_perez  15.3M  12.3G   306K  43.8K 20150813 4 min 54 s
    fresh_water_scarcity_an_introduction_to_the_problem_christia  4.98M  12.3G   392K  43.6K 20130214 3 min 38 s
                      attack_of_the_killer_algae_eric_noel_munoz  7.21M  12.3G   348K  43.5K 20140624 3 min 23 s
                  three_months_after_by_cristin_o_keefe_aptowicz  2.85M  12.4G   130K  43.2K 20190202 1 min 24 s
               the_poet_who_painted_with_his_words_genevieve_emy  8.16M  12.4G   258K  43.0K 20160321 4 min 15 s
    the_fundamental_theorem_of_arithmetic_computer_science_khan_  8.53M  12.4G   429K  42.9K 20120327 3 min 51 s
    a_needle_in_countless_haystacks_finding_habitable_worlds_ari  15.8M  12.4G   386K  42.9K 20121108 5 min 10 s
    how_does_an_atom_smashing_particle_accelerator_work_don_linc  6.99M  12.4G   385K  42.8K 20130418 3 min 35 s
          an_introduction_to_mathematical_theorems_scott_kennedy  9.47M  12.4G   385K  42.7K 20120910 4 min 38 s
    how_spontaneous_brain_activity_keeps_you_alive_nathan_s_jaco  9.08M  12.4G   298K  42.6K 20150113 5 min 17 s
    how_science_fiction_can_help_predict_the_future_roey_tzezana  17.6M  12.4G   254K  42.4K 20160126 5 min 21 s
    forget_shopping_soon_you_ll_download_your_new_clothes_danit_  16.3M  12.4G   254K  42.4K 20151217 6 min 22 s
                            the_world_s_english_mania_jay_walker  10.2M  12.5G   376K  41.7K 20130301 4 min 31 s
         cicadas_the_dormant_army_beneath_your_feet_rose_eveleth  5.00M  12.5G   375K  41.7K 20130905 2 min 45 s
                           gravity_and_the_human_body_jay_buckey  7.82M  12.5G   375K  41.7K 20130826 4 min 45 s
    how_to_speak_monkey_the_language_of_cotton_top_tamarins_anne  11.5M  12.5G   333K  41.6K 20140626 5 min 13 s
    haptography_digitizing_our_sense_of_touch_katherine_kuchenbe  13.9M  12.5G   373K  41.4K 20130325 6 min 28 s
     the_science_of_macaroni_salad_what_s_in_a_mixture_josh_kurz  7.48M  12.5G   369K  41.0K 20130731 3 min 56 s
    a_poetic_experiment_walt_whitman_interpreted_by_three_animat  10.1M  12.5G   287K  41.0K 20150820 3 min 28 s
               how_to_turn_protest_into_powerful_change_eric_liu  10.0M  12.5G   242K  40.4K 20160714 4 min 56 s
                                             ted_ed_website_tour  7.12M  12.5G   401K  40.1K 20120425 3 min 7 s
    lessons_from_auschwitz_the_power_of_our_words_benjamin_zande  3.24M  12.5G   318K  39.8K 20140425 1 min 20 s
                         learn_to_read_chinese_with_ease_shaolan  8.99M  12.5G   318K  39.7K 20131203 6 min 14 s
                the_invisible_motion_of_still_objects_ran_tivony  10.6M  12.5G   238K  39.6K 20160324 4 min 43 s
                     how_algorithms_shape_our_world_kevin_slavin  37.2M  12.6G   357K  39.6K 20121125 15 min 23 s
                   shakespearean_dating_tips_anthony_john_peters  4.48M  12.6G   356K  39.6K 20130822 2 min 24 s
                       the_case_against_good_and_bad_marlee_neel  10.4M  12.6G   396K  39.6K 20120709 4 min 52 s
                       could_a_blind_eye_regenerate_david_davila  8.95M  12.6G   266K  38.0K 20150115 4 min 6 s
    illuminating_photography_from_camera_obscura_to_camera_phone  7.92M  12.6G   341K  37.9K 20130228 4 min 49 s
                               disappearing_frogs_kerry_m_kriger  5.68M  12.6G   301K  37.6K 20130916 3 min 47 s
                               why_do_americans_vote_on_tuesdays  7.96M  12.6G   375K  37.5K 20120410 3 min 27 s
                    how_containerization_shaped_the_modern_world  11.4M  12.6G   374K  37.4K 20120311 4 min 46 s
              what_cameras_see_that_our_eyes_don_t_bill_shribman  6.84M  12.6G   336K  37.3K 20130410 3 min 19 s
                                how_bacteria_talk_bonnie_bassler  40.1M  12.7G   335K  37.2K 20130209 18 min 11 s
     what_did_dogs_teach_humans_about_diabetes_duncan_c_ferguson  6.39M  12.7G   295K  36.9K 20140828 3 min 47 s
    why_do_americans_and_canadians_celebrate_labor_day_kenneth_c  13.1M  12.7G   328K  36.4K 20130830 4 min 12 s
                     ideasthesia_how_do_ideas_feel_danko_nikolic  8.36M  12.7G   254K  36.2K 20141106 5 min 37 s
                    the_best_stats_you_ve_ever_seen_hans_rosling  36.0M  12.7G   322K  35.8K 20130713 19 min 53 s
                               how_plants_tell_time_dasha_savage  7.16M  12.8G   250K  35.7K 20150611 4 min 19 s
           under_the_hood_the_chemistry_of_cars_cynthia_chubbuck  6.16M  12.8G   282K  35.3K 20140724 4 min 33 s
               where_we_get_our_fresh_water_christiana_z_peppard  6.66M  12.8G   315K  35.0K 20130212 3 min 46 s
                          rapid_prototyping_google_glass_tom_chi  19.1M  12.8G   314K  34.9K 20130122 8 min 8 s
                           the_twisting_tale_of_dna_judith_hauck  12.6M  12.8G   314K  34.9K 20121003 4 min 26 s
           how_does_math_guide_our_ships_at_sea_george_christoph  9.91M  12.8G   309K  34.4K 20121011 4 min 18 s
                            the_time_value_of_money_german_nande  5.93M  12.8G   268K  33.5K 20140703 3 min 36 s
                                   where_did_the_earth_come_from  10.5M  12.8G   367K  33.4K 20110512 3 min 56 s
                                 how_do_nerves_work_elliot_krane  11.9M  12.8G   333K  33.3K 20120809 4 min 59 s
                       is_space_trying_to_kill_us_ron_shaneyfelt  6.67M  12.8G   295K  32.8K 20130528 3 min 30 s
                the_cancer_gene_we_all_have_michael_windelspecht  6.82M  12.8G   262K  32.7K 20140519 3 min 18 s
                           the_great_brain_debate_ted_altschuler  13.4M  12.9G   227K  32.5K 20141117 5 min 19 s
    all_of_the_energy_in_the_universe_is_george_zaidan_and_charl  10.7M  12.9G   259K  32.3K 20131112 3 min 51 s
                             making_sense_of_spelling_gina_cooke  6.93M  12.9G   289K  32.1K 20120925 4 min 18 s
    from_aaliyah_to_jay_z_captured_moments_in_hip_hop_history_jo  12.2M  12.9G   254K  31.7K 20140515 6 min 20 s
         let_s_make_history_by_recording_it_storycorps_ted_prize  7.31M  12.9G   190K  31.6K 20151123 3 min 17 s
                               how_life_begins_in_the_deep_ocean  18.1M  12.9G   316K  31.6K 20120514 6 min 1 s
          what_we_can_learn_from_galaxies_far_far_away_henry_lin  13.7M  12.9G   247K  30.8K 20140227 6 min 43 s
               the_world_needs_all_kinds_of_minds_temple_grandin  48.2M  13.0G   277K  30.8K 20130210 19 min 43 s
          the_history_of_our_world_in_18_minutes_david_christian  35.7M  13.0G   274K  30.5K 20130315 17 min 40 s
                         how_to_think_about_gravity_jon_bergmann  8.26M  13.0G   273K  30.3K 20120917 4 min 43 s
    the_hidden_worlds_within_natural_history_museums_joshua_drew  7.12M  13.0G   212K  30.2K 20141202 4 min 26 s
         what_light_can_teach_us_about_the_universe_pete_edwards  10.5M  13.0G   237K  29.6K 20140731 4 min 6 s
          rnai_slicing_dicing_and_serving_your_cells_alex_dainis  7.38M  13.0G   263K  29.2K 20130812 4 min 7 s
                              the_carbon_cycle_nathaniel_manning  9.35M  13.1G   261K  29.0K 20121002 3 min 54 s
    what_is_the_shape_of_a_molecule_george_zaidan_and_charles_mo  5.74M  13.1G   232K  29.0K 20131017 3 min 47 s
                how_the_language_you_speak_affects_your_thoughts  16.7M  13.1G   143K  28.7K 20170427 8 min 45 s
                 poetic_stickup_put_the_financial_aid_in_the_bag  11.9M  13.1G   286K  28.6K 20120321 5 min 4 s
                                                dear_subscribers  8.46M  13.1G   255K  28.4K 20130319 2 min 42 s
    diagnosing_a_zombie_brain_and_behavior_part_two_tim_verstyne  6.01M  13.1G   253K  28.1K 20121024 3 min 43 s
                                how_breathing_works_nirvair_kaur  10.1M  13.1G   253K  28.1K 20121004 5 min 18 s
                would_you_weigh_less_in_an_elevator_carol_hedden  8.40M  13.1G   253K  28.1K 20121119 3 min 35 s
                 et_is_probably_out_there_get_ready_seth_shostak  31.7M  13.1G   252K  28.0K 20130821 18 min 40 s
     a_curable_condition_that_causes_blindness_andrew_bastawrous  5.69M  13.2G   167K  27.8K 20150928 4 min 22 s
             the_nurdles_quest_for_ocean_domination_kim_preshoff  9.76M  13.2G   221K  27.6K 20140804 4 min 54 s
                      who_controls_the_world_james_b_glattfelder  25.8M  13.2G   245K  27.3K 20130515 14 min 10 s
                               let_s_talk_about_dying_peter_saul  22.7M  13.2G   243K  27.0K 20130609 13 min 19 s
                     the_secret_lives_of_baby_fish_amy_mcdermott  10.4M  13.2G   212K  26.5K 20140811 3 min 58 s
                     who_is_alexander_von_humboldt_george_mehler  9.36M  13.2G   238K  26.4K 20130402 4 min 21 s
                     india_s_invisible_innovation_nirmalya_kumar  24.6M  13.3G   235K  26.1K 20130821 15 min 12 s
                        why_i_m_a_weekday_vegetarian_graham_hill  8.79M  13.3G   233K  25.9K 20130222 4 min 4 s
                    making_a_ted_ed_lesson_visualizing_big_ideas  10.7M  13.3G   207K  25.9K 20131125 5 min 3 s
         if_superpowers_were_real_which_would_you_choose_joy_lin  6.26M  13.3G   231K  25.6K 20130627 2 min 21 s
                        how_to_detect_a_supernova_samantha_kuula  10.5M  13.3G   178K  25.4K 20150609 4 min 41 s
    symbiosis_a_surprising_tale_of_species_cooperation_david_gon  6.18M  13.3G   251K  25.1K 20120313 2 min 22 s
                               how_i_discovered_dna_james_watson  34.2M  13.3G   223K  24.8K 20130726 20 min 14 s
        want_to_be_an_activist_start_with_your_toys_mckenna_pope  15.9M  13.3G   197K  24.6K 20140129 5 min 21 s
                           how_movies_teach_manhood_colin_stokes  23.3M  13.4G   219K  24.3K 20130522 12 min 56 s
    what_is_chirality_and_how_did_it_get_in_my_molecules_michael  8.29M  13.4G   217K  24.1K 20120920 5 min 4 s
    from_dna_to_silly_putty_the_diverse_world_of_polymers_jan_ma  13.1M  13.4G   192K  24.0K 20131210 4 min 59 s
                                how_does_work_work_peter_bohacek  13.5M  13.4G   215K  23.9K 20121129 4 min 30 s
      mysteries_of_vernacular_odd_jessica_oreck_and_rachael_teel  2.67M  13.4G   190K  23.7K 20131205 1 min 51 s
                     the_game_changing_amniotic_egg_april_tucker  10.7M  13.4G   208K  23.2K 20130618 4 min 29 s
    beach_bodies_in_spoken_word_david_fasanya_and_gabriel_barral  11.1M  13.4G   207K  23.0K 20130227 3 min 32 s
    equality_sports_and_title_ix_erin_buzuvis_and_kristine_newha  9.10M  13.4G   205K  22.8K 20130619 4 min 34 s
    is_our_climate_headed_for_a_mathematical_tipping_point_victo  8.10M  13.4G   158K  22.6K 20141023 4 min 10 s
    making_waves_the_power_of_concentration_gradients_sasha_wrig  9.82M  13.5G   178K  22.3K 20140304 5 min 19 s
    the_surprising_and_invisible_signatures_of_sea_creatures_kak  19.3M  13.5G   133K  22.2K 20160107 6 min 37 s
    cloudy_climate_change_how_clouds_affect_earth_s_temperature_  19.7M  13.5G   154K  22.0K 20140925 6 min 39 s
                                a_host_of_heroes_april_gudenrath  13.9M  13.5G   197K  21.9K 20130429 4 min 53 s
              mining_literature_for_deeper_meanings_amy_e_harter  7.89M  13.5G   197K  21.9K 20130531 4 min 12 s
    the_science_of_macaroni_salad_what_s_in_a_molecule_josh_kurz  7.34M  13.5G   194K  21.6K 20130816 3 min 14 s
                      the_key_to_media_s_hidden_codes_ben_beaton  19.2M  13.5G   214K  21.4K 20120529 5 min 59 s
                                   the_tribes_we_lead_seth_godin  34.8M  13.6G   191K  21.2K 20130303 17 min 26 s
    why_the_arctic_is_climate_change_s_canary_in_the_coal_mine_w  7.75M  13.6G   148K  21.1K 20150122 3 min 58 s
     the_oddities_of_the_first_american_election_kenneth_c_davis  14.8M  13.6G   188K  20.9K 20121105 4 min 6 s
                    write_your_story_change_history_brad_meltzer  14.9M  13.6G   186K  20.7K 20130116 8 min 57 s
                                         beatboxing_101_beat_nyc  20.4M  13.6G   186K  20.7K 20130702 6 min 8 s
                                how_to_fool_a_gps_todd_humphreys  29.0M  13.7G   184K  20.4K 20130626 15 min 45 s
            making_sense_of_how_life_fits_together_bobbi_seleski  7.42M  13.7G   184K  20.4K 20130423 4 min 52 s
    how_cosmic_rays_help_us_understand_the_universe_veronica_bin  16.4M  13.7G   142K  20.2K 20140923 4 min 39 s
                   how_social_media_can_make_history_clay_shirky  31.6M  13.7G   181K  20.1K 20121116 15 min 48 s
                                       start_a_ted_ed_club_today  4.62M  13.7G   160K  20.0K 20140114 2 min 21 s
                    how_to_stop_being_bored_and_start_being_bold  29.8M  13.7G  79.9K  20.0K 20180126 9 min 57 s
                bird_migration_a_perilous_journey_alyssa_klavans  7.10M  13.7G   159K  19.9K 20130917 4 min 9 s
                    what_adults_can_learn_from_kids_adora_svitak  16.5M  13.8G   176K  19.6K 20130222 8 min 12 s
    insights_into_cell_membranes_via_dish_detergent_ethan_perlst  6.72M  13.8G   176K  19.6K 20130226 3 min 49 s
            what_if_we_could_look_inside_human_brains_moran_cerf  13.9M  13.8G   175K  19.5K 20130131 3 min 55 s
        pros_and_cons_of_public_opinion_polls_jason_robert_jaffe  6.46M  13.8G   174K  19.3K 20130517 4 min 24 s
                      free_falling_in_outer_space_matt_j_carlson  4.53M  13.8G   173K  19.2K 20130706 2 min 58 s
                              the_power_of_introverts_susan_cain  35.7M  13.8G   173K  19.2K 20121225 19 min 3 s
                                       network_theory_marc_samet  6.86M  13.8G   171K  19.0K 20130123 3 min 30 s
                             stroke_of_insight_jill_bolte_taylor  44.4M  13.9G   169K  18.7K 20121114 18 min 41 s
    how_did_trains_standardize_time_in_the_united_states_william  7.29M  13.9G   167K  18.6K 20130205 3 min 34 s
               the_real_origin_of_the_franchise_sir_harold_evans  16.9M  13.9G   185K  18.5K 20120326 5 min 48 s
            networking_for_the_networking_averse_lisa_green_chau  7.31M  13.9G   166K  18.5K 20130403 3 min 30 s
       rhythm_in_a_box_the_story_of_the_cajon_drum_paul_jennings  10.6M  13.9G   126K  18.0K 20150303 3 min 29 s
    activation_energy_kickstarting_chemical_reactions_vance_kite  8.80M  13.9G   157K  17.5K 20130109 3 min 22 s
                       your_brain_on_video_games_daphne_bavelier  28.8M  14.0G   156K  17.3K 20130327 17 min 57 s
    speech_acts_constative_and_performative_colleen_glenney_bogg  9.02M  14.0G   138K  17.2K 20131003 3 min 57 s
                       the_history_of_keeping_time_karen_mensing  9.31M  14.0G   169K  16.9K 20120816 3 min 47 s
                            sunflowers_and_fibonacci_numberphile  17.5M  14.0G   169K  16.9K 20120410 5 min 25 s
             how_bees_help_plants_have_sex_fernanda_s_valdovinos  8.59M  14.0G   135K  16.8K 20140617 5 min 25 s
                               what_on_earth_is_spin_brian_jones  12.1M  14.0G   148K  16.5K 20130529 3 min 56 s
                     the_happy_secret_to_better_work_shawn_achor  23.2M  14.0G   147K  16.4K 20130616 12 min 20 s
         could_comets_be_the_source_of_life_on_earth_justin_dowd  7.00M  14.0G   114K  16.3K 20141028 3 min 37 s
          animation_basics_the_optical_illusion_of_motion_ted_ed  14.6M  14.1G   147K  16.3K 20130713 5 min 11 s
                      mosquitos_malaria_and_education_bill_gates  61.1M  14.1G   146K  16.2K 20130222 20 min 20 s
                   slowing_down_time_in_writing_film_aaron_sitze  13.8M  14.1G   146K  16.2K 20130124 5 min 59 s
            the_magic_of_qr_codes_in_the_classroom_karen_mensing  8.90M  14.1G   143K  15.9K 20130620 5 min 17 s
                              how_life_came_to_land_tierney_thys  17.8M  14.2G   142K  15.7K 20121107 5 min 27 s
               inventing_the_american_presidency_kenneth_c_davis  9.96M  14.2G   140K  15.6K 20130218 4 min 0 s
       silk_the_ancient_material_of_the_future_fiorenzo_omenetto  15.9M  14.2G   139K  15.4K 20130322 9 min 40 s
                        my_glacier_cave_discoveries_eddy_cartaya  16.7M  14.2G   123K  15.3K 20131211 8 min 2 s
           the_abc_s_of_gas_avogadro_boyle_charles_brian_bennett  6.01M  14.2G   138K  15.3K 20121009 2 min 49 s
      a_trip_through_space_to_calculate_distance_heather_tunnell  6.01M  14.2G   137K  15.2K 20130906 3 min 46 s
            why_you_will_fail_to_have_a_great_career_larry_smith  41.2M  14.3G   137K  15.2K 20130815 15 min 15 s
                animation_basics_homemade_special_effects_ted_ed  10.4M  14.3G   134K  14.9K 20130516 4 min 18 s
     mysteries_of_vernacular_lady_jessica_oreck_and_rachael_teel  2.94M  14.3G   119K  14.9K 20131121 2 min 7 s
              euclid_s_puzzling_parallel_postulate_jeff_dekofsky  6.13M  14.3G   132K  14.7K 20130326 3 min 36 s
     let_s_talk_about_sex_john_bohannon_and_black_label_movement  30.3M  14.3G   130K  14.5K 20121203 10 min 42 s
           it_s_time_to_question_bio_engineering_paul_root_wolpe  29.3M  14.3G   130K  14.5K 20130815 19 min 42 s
                                         evolution_in_a_big_city  11.5M  14.3G   144K  14.4K 20120311 5 min 15 s
                                  adhd_finding_what_works_for_me  26.0M  14.4G  57.5K  14.4K 20180223 12 min 1 s
    why_extremophiles_bode_well_for_life_beyond_earth_louisa_pre  7.56M  14.4G   112K  14.1K 20131007 3 min 59 s
                           printing_a_human_kidney_anthony_atala  39.3M  14.4G   126K  14.0K 20130315 16 min 54 s
      how_farming_planted_seeds_for_the_internet_patricia_russac  6.56M  14.4G   122K  13.6K 20130321 3 min 58 s
                   how_to_take_a_great_picture_carolina_molinari  4.93M  14.4G   122K  13.6K 20130729 2 min 57 s
                           don_t_insist_on_english_patricia_ryan  16.4M  14.4G   122K  13.6K 20130725 10 min 35 s
                  your_elusive_creative_genius_elizabeth_gilbert  40.2M  14.5G   121K  13.5K 20130322 19 min 31 s
    string_theory_and_the_hidden_structures_of_the_universe_clif  15.7M  14.5G   120K  13.4K 20130422 7 min 52 s
                                 tales_of_passion_isabel_allende  29.3M  14.5G   120K  13.3K 20130106 18 min 30 s
    the_mysterious_workings_of_the_adolescent_brain_sarah_jayne_  31.1M  14.6G   116K  12.9K 20130601 14 min 26 s
        want_to_be_happier_stay_in_the_moment_matt_killingsworth  18.3M  14.6G   115K  12.8K 20130329 10 min 16 s
                                  cern_s_supercollider_brian_cox  26.8M  14.6G   115K  12.8K 20121208 14 min 56 s
                      greeting_the_world_in_peace_jackie_jenkins  9.34M  14.6G   124K  12.4K 20120904 3 min 17 s
                       inside_a_cartoonist_s_world_liza_donnelly  12.4M  14.6G   112K  12.4K 20121211 4 min 22 s
                  curiosity_discovery_and_gecko_feet_robert_full  13.7M  14.6G   111K  12.4K 20121220 9 min 9 s
                           why_do_we_see_illusions_mark_changizi  15.5M  14.6G   111K  12.4K 20130320 7 min 21 s
                       the_story_behind_your_glasses_eva_timothy  12.4M  14.7G   111K  12.3K 20121008 4 min 16 s
        describing_the_invisible_properties_of_gas_brian_bennett  7.12M  14.7G   111K  12.3K 20121010 3 min 25 s
     the_emergence_of_drama_as_a_literary_art_mindy_ploeckelmann  7.41M  14.7G   111K  12.3K 20130605 3 min 46 s
       pavlovian_reactions_aren_t_just_for_dogs_benjamin_n_witts  12.0M  14.7G   110K  12.3K 20130426 3 min 53 s
                     an_exercise_in_time_perception_matt_danzico  10.3M  14.7G   110K  12.2K 20130611 5 min 24 s
               strange_answers_to_the_psychopath_test_jon_ronson  32.7M  14.7G   110K  12.2K 20130507 18 min 1 s
     magical_metals_how_shape_memory_alloys_work_ainissa_ramirez  10.0M  14.7G   109K  12.1K 20120918 4 min 45 s
    mysteries_of_vernacular_robot_jessica_oreck_and_rachael_teel  3.26M  14.7G  92.2K  11.5K 20131031 2 min 17 s
          distant_time_and_the_hint_of_a_multiverse_sean_carroll  26.5M  14.8G   103K  11.4K 20130808 15 min 54 s
                a_digital_reimagining_of_gettysburg_anne_knowles  15.6M  14.8G  90.7K  11.3K 20140526 9 min 3 s
                  the_networked_beauty_of_forests_suzanne_simard  22.0M  14.8G  88.6K  11.1K 20140414 7 min 23 s
                       how_photography_connects_us_david_griffin  21.2M  14.8G  98.1K  10.9K 20130101 14 min 56 s
                  distorting_madonna_in_medieval_art_james_earle  5.58M  14.8G  97.8K  10.9K 20130219 3 min 10 s
        parasite_tales_the_jewel_wasp_s_zombie_slave_carl_zimmer  15.8M  14.8G  97.6K  10.8K 20130128 7 min 11 s
                   science_can_answer_moral_questions_sam_harris  47.2M  14.9G  97.4K  10.8K 20130216 23 min 37 s
                     does_racism_affect_how_you_vote_nate_silver  15.5M  14.9G  97.2K  10.8K 20130302 9 min 13 s
                   actually_the_world_isn_t_flat_pankaj_ghemawat  41.9M  14.9G  95.7K  10.6K 20130525 17 min 3 s
                          on_positive_psychology_martin_seligman  42.8M  15.0G  95.3K  10.6K 20130706 23 min 45 s
                   pruney_fingers_a_gripping_story_mark_changizi  9.86M  15.0G  93.9K  10.4K 20130521 4 min 21 s
                                              is_equality_enough  14.0M  15.0G  51.8K  10.4K 20170720 7 min 46 s
                       see_yemen_through_my_eyes_nadia_al_sakkaf  26.1M  15.0G  93.2K  10.4K 20121226 13 min 38 s
                           are_video_games_actually_good_for_you  13.5M  15.0G  51.7K  10.3K 20170518 8 min 18 s
                        all_your_devices_can_be_hacked_avi_rubin  24.9M  15.1G  92.5K  10.3K 20130612 16 min 56 s
                                a_tap_dancer_s_craft_andrew_nemr  13.4M  15.1G  91.8K  10.2K 20121219 5 min 50 s
      dark_matter_how_does_it_explain_a_star_s_speed_don_lincoln  5.78M  15.1G  91.0K  10.1K 20121112 3 min 16 s
                               connected_but_alone_sherry_turkle  34.2M  15.1G  91.0K  10.1K 20130419 19 min 48 s
       the_weird_wonderful_world_of_bioluminescence_edith_widder  20.8M  15.1G  90.9K  10.1K 20130329 12 min 45 s
    defining_cyberwarfare_in_hopes_of_preventing_it_daniel_garri  7.67M  15.2G  90.5K  10.1K 20130820 3 min 49 s
                        the_mystery_of_chronic_pain_elliot_krane  16.0M  15.2G  90.2K  10.0K 20130331 8 min 14 s
                    america_s_native_prisoners_of_war_aaron_huey  24.2M  15.2G  79.6K  9.95K 20130913 15 min 27 s
                           math_class_needs_a_makeover_dan_meyer  18.7M  15.2G  89.5K  9.94K 20130801 11 min 39 s
                   what_s_wrong_with_our_food_system_birke_baehr  8.37M  15.2G  89.2K  9.91K 20130816 5 min 14 s
                       how_i_learned_to_organize_my_scatterbrain  30.4M  15.2G  39.6K  9.91K 20180419 13 min 4 s
    put_those_smartphones_away_great_tips_for_making_your_job_in  19.0M  15.3G  88.6K  9.84K 20130528 6 min 21 s
                          the_power_of_vulnerability_brene_brown  32.3M  15.3G  88.2K  9.80K 20130710 20 min 19 s
            the_neurons_that_shaped_civilization_vs_ramachandran  13.9M  15.3G  87.7K  9.74K 20130831 7 min 43 s
                            underwater_astonishments_david_gallo  13.3M  15.3G  87.7K  9.74K 20130111 5 min 24 s
                              sending_a_sundial_to_mars_bill_nye  13.0M  15.3G  87.1K  9.68K 20121016 7 min 49 s
                    how_great_leaders_inspire_action_simon_sinek  55.7M  15.4G  86.8K  9.64K 20130626 18 min 4 s
                     taking_imagination_seriously_janet_echelman  27.4M  15.4G  86.7K  9.63K 20130811 10 min 29 s
                                the_future_of_lying_jeff_hancock  54.2M  15.5G  85.9K  9.54K 20130403 18 min 31 s
                   capturing_authentic_narratives_michele_weldon  7.14M  15.5G  93.3K  9.33K 20120827 3 min 18 s
    ted_ed_clubs_celebrating_and_amplifying_student_voices_aroun  5.79M  15.5G  46.5K  9.31K 20170301 2 min 12 s
    mysteries_of_vernacular_yankee_jessica_oreck_and_rachael_tee  2.83M  15.5G  74.1K  9.27K 20131126 1 min 54 s
       a_call_to_invention_diy_speaker_edition_william_gurstelle  21.0M  15.5G  83.4K  9.27K 20130404 6 min 48 s
               let_s_use_video_to_reinvent_education_salman_khan  36.3M  15.5G  83.1K  9.23K 20130316 20 min 27 s
                             how_to_track_a_tornado_karen_kosiba  14.1M  15.6G  73.2K  9.15K 20140421 5 min 44 s
                                making_a_ted_ed_lesson_animation  13.2M  15.6G  82.1K  9.12K 20130527 5 min 7 s
          biofuels_and_bioprospecting_for_beginners_craig_a_kohn  9.67M  15.6G  81.5K  9.05K 20130513 3 min 52 s
                          on_exploring_the_oceans_robert_ballard  32.3M  15.6G  80.2K  8.91K 20130105 18 min 16 s
          how_two_decisions_led_me_to_olympic_glory_steve_mesler  9.62M  15.6G  88.6K  8.86K 20120730 4 min 1 s
               a_teen_just_trying_to_figure_it_out_tavi_gevinson  13.2M  15.6G  78.8K  8.76K 20130821 7 min 30 s
          gridiron_physics_scalars_and_vectors_michelle_buchanan  10.8M  15.6G  78.7K  8.74K 20130220 4 min 48 s
                measuring_what_makes_life_worthwhile_chip_conley  30.8M  15.7G  77.8K  8.64K 20121227 17 min 39 s
                   hiv_and_flu_the_vaccine_strategy_seth_berkley  48.5M  15.7G  77.2K  8.58K 20130308 21 min 5 s
                      deep_sea_diving_in_a_wheelchair_sue_austin  26.6M  15.7G  68.5K  8.57K 20131211 9 min 38 s
                cleaning_our_oceans_a_big_plan_for_a_big_problem  27.3M  15.8G  33.2K  8.30K 20180309 10 min 52 s
               what_makes_us_feel_good_about_our_work_dan_ariely  31.8M  15.8G  65.9K  8.24K 20131206 20 min 26 s
    conserving_our_spectacular_vulnerable_coral_reefs_joshua_dre  7.22M  15.8G  73.4K  8.16K 20130103 3 min 14 s
                             on_spaghetti_sauce_malcolm_gladwell  40.2M  15.9G  73.3K  8.14K 20130706 17 min 33 s
                        questioning_the_universe_stephen_hawking  19.1M  15.9G  72.7K  8.08K 20130114 10 min 15 s
                         the_sweaty_teacher_s_lament_justin_lamb  9.95M  15.9G  64.4K  8.05K 20140506 3 min 10 s
     mysteries_of_vernacular_zero_jessica_oreck_and_rachael_teel  2.82M  15.9G  72.4K  8.04K 20130801 2 min 5 s
                               our_loss_of_wisdom_barry_schwartz  63.2M  15.9G  72.1K  8.01K 20130202 21 min 19 s
              breaking_the_illusion_of_skin_color_nina_jablonski  26.8M  16.0G  71.7K  7.97K 20121224 14 min 45 s
                                              math_is_everywhere  8.18M  16.0G  31.7K  7.93K 20180227 4 min 16 s
    mysteries_of_vernacular_ukulele_jessica_oreck_and_rachael_te  2.69M  16.0G  63.3K  7.92K 20131018 2 min 0 s
             dance_vs_powerpoint_a_modest_proposal_john_bohannon  31.7M  16.0G  69.5K  7.72K 20121128 11 min 17 s
                     why_work_doesn_t_happen_at_work_jason_fried  24.0M  16.0G  69.4K  7.71K 20130710 15 min 20 s
     the_family_structure_of_elephants_caitlin_o_connell_rodwell  16.6M  16.1G  61.1K  7.64K 20140415 8 min 11 s
                   the_linguistic_genius_of_babies_patricia_kuhl  15.9M  16.1G  67.7K  7.52K 20130710 10 min 17 s
         how_to_find_the_true_face_of_leonardo_siegfried_woldhek  6.53M  16.1G  67.2K  7.46K 20130113 4 min 21 s
              we_need_to_talk_about_an_injustice_bryan_stevenson  41.9M  16.1G  67.0K  7.44K 20130412 23 min 41 s
                        your_genes_are_not_your_fate_dean_ornish  4.86M  16.1G  66.8K  7.42K 20130117 3 min 14 s
       dissecting_botticelli_s_adoration_of_the_magi_james_earle  7.88M  16.1G  66.6K  7.40K 20130617 3 min 11 s
                             historical_role_models_amy_bissetta  5.19M  16.1G  66.2K  7.35K 20130507 2 min 36 s
                                i_listen_to_color_neil_harbisson  24.6M  16.2G  65.6K  7.28K 20130621 9 min 36 s
                     how_curiosity_got_us_to_mars_bobak_ferdowsi  12.3M  16.2G  65.5K  7.28K 20130211 6 min 13 s
          phenology_and_nature_s_shifting_rhythms_regina_brinker  6.93M  16.2G  64.7K  7.19K 20130110 3 min 41 s
                                  listening_to_shame_brene_brown  40.4M  16.2G  64.7K  7.19K 20130719 20 min 38 s
    could_your_language_affect_your_ability_to_save_money_keith_  25.0M  16.2G  56.7K  7.09K 20131203 12 min 12 s
                      mysteries_of_vernacular_clue_jessica_oreck  3.72M  16.2G  63.4K  7.04K 20130401 1 min 58 s
                             folding_way_new_origami_robert_lang  30.5M  16.3G  62.4K  6.94K 20121209 15 min 56 s
                        making_a_ted_ed_lesson_animating_zombies  9.24M  16.3G  62.0K  6.88K 20130714 5 min 6 s
                  how_to_make_work_life_balance_work_nigel_marsh  21.7M  16.3G  61.6K  6.85K 20130808 10 min 4 s
    self_assembly_the_power_of_organizing_the_unorganized_skylar  6.34M  16.3G  60.4K  6.71K 20130408 3 min 41 s
    how_one_teenager_unearthed_baseball_s_untold_history_cam_per  13.5M  16.3G  53.4K  6.68K 20140206 5 min 55 s
    how_whales_breathe_communicate_and_fart_with_their_faces_joy  16.1M  16.3G  53.3K  6.67K 20140410 6 min 24 s
             3_things_i_learned_while_my_plane_crashed_ric_elias  9.01M  16.3G  59.0K  6.55K 20130315 5 min 2 s
                               make_robots_smarter_ayanna_howard  19.2M  16.4G  58.1K  6.45K 20130206 6 min 18 s
                    the_danger_of_science_denial_michael_specter  36.6M  16.4G  57.9K  6.43K 20130223 16 min 29 s
    how_giant_sea_creatures_eat_tiny_sea_creatures_kelly_benoit_  13.0M  16.4G  57.9K  6.43K 20130509 6 min 21 s
                  visualizing_the_world_s_twitter_data_jer_thorp  14.6M  16.4G  57.9K  6.43K 20130221 5 min 41 s
                                why_is_x_the_unknown_terry_moore  6.43M  16.4G  57.8K  6.43K 20130507 3 min 57 s
    mysteries_of_vernacular_quarantine_jessica_oreck_and_rachael  3.45M  16.4G  57.5K  6.39K 20130704 2 min 10 s
                     stories_legacies_of_who_we_are_awele_makeba  20.8M  16.5G  63.3K  6.33K 20120312 9 min 1 s
       is_the_obesity_crisis_hiding_a_bigger_problem_peter_attia  38.9M  16.5G  56.7K  6.30K 20130823 16 min 1 s
                  tracking_grizzly_bears_from_space_david_laskin  11.5M  16.5G  56.5K  6.27K 20130604 4 min 14 s
              learning_from_past_presidents_doris_kearns_goodwin  29.6M  16.5G  56.4K  6.26K 20130121 19 min 16 s
                               a_plant_s_eye_view_michael_pollan  45.5M  16.6G  55.7K  6.19K 20130112 17 min 29 s
                         the_human_and_the_honeybee_dino_martins  13.5M  16.6G  55.1K  6.12K 20130629 6 min 24 s
         early_forensics_and_crime_solving_chemists_deborah_blum  15.9M  16.6G  54.8K  6.09K 20130401 7 min 50 s
                           less_stuff_more_happiness_graham_hill  8.50M  16.6G  54.6K  6.06K 20130412 5 min 49 s
                  mysteries_of_vernacular_assassin_jessica_oreck  3.34M  16.6G  54.4K  6.05K 20130403 1 min 56 s
               the_pattern_behind_self_deception_michael_shermer  32.1M  16.7G  53.3K  5.92K 20130308 19 min 1 s
                how_architecture_helped_music_evolve_david_byrne  31.4M  16.7G  53.1K  5.90K 20130308 16 min 0 s
                             averting_the_climate_crisis_al_gore  48.8M  16.7G  52.8K  5.87K 20130510 16 min 14 s
    mysteries_of_vernacular_bewilder_jessica_oreck_and_rachael_t  3.08M  16.7G  52.7K  5.85K 20130823 1 min 54 s
            how_art_gives_shape_to_cultural_change_thelma_golden  19.5M  16.8G  52.5K  5.83K 20130224 12 min 28 s
                time_lapse_proof_of_extreme_ice_loss_james_balog  45.5M  16.8G  51.8K  5.76K 20130817 19 min 19 s
    mysteries_of_vernacular_earwig_jessica_oreck_and_rachael_tee  3.50M  16.8G  51.7K  5.75K 20130510 2 min 15 s
       ashton_cofer_a_young_inventor_s_plan_to_recycle_styrofoam  11.2M  16.8G  28.7K  5.73K 20170420 6 min 5 s
           visualizing_hidden_worlds_inside_your_body_dee_breger  12.0M  16.8G  51.1K  5.67K 20130306 6 min 5 s
                          a_40_year_plan_for_energy_amory_lovins  41.3M  16.9G  50.3K  5.58K 20130427 27 min 4 s
                          the_hidden_power_of_smiling_ron_gutman  15.4M  16.9G  49.6K  5.52K 20130322 7 min 26 s
          why_i_must_speak_out_about_climate_change_james_hansen  28.7M  16.9G  49.6K  5.51K 20130412 17 min 51 s
                         different_ways_of_knowing_daniel_tammet  16.6M  16.9G  49.3K  5.47K 20130405 10 min 53 s
                              earth_s_mass_extinction_peter_ward  29.1M  16.9G  49.1K  5.46K 20130201 19 min 38 s
                                        true_success_john_wooden  53.9M  17.0G  49.0K  5.45K 20130824 17 min 39 s
            how_state_budgets_are_breaking_us_schools_bill_gates  18.3M  17.0G  48.8K  5.42K 20130309 11 min 31 s
                               click_your_fortune_episode_1_demo  5.03M  17.0G  47.1K  5.24K 20130807 3 min 23 s
                                 the_bottom_billion_paul_collier  37.6M  17.1G  47.0K  5.22K 20130116 16 min 52 s
    mysteries_of_vernacular_gorgeous_jessica_oreck_and_rachael_t  3.17M  17.1G  46.8K  5.20K 20130621 2 min 0 s
            making_a_ted_ed_lesson_synesthesia_and_playing_cards  9.15M  17.1G  46.7K  5.19K 20130707 4 min 6 s
                                     the_birth_of_a_word_deb_roy  43.2M  17.1G  46.7K  5.19K 20121201 21 min 7 s
              high_altitude_wind_energy_from_kites_saul_griffith  11.2M  17.1G  46.6K  5.18K 20130222 5 min 22 s
    what_we_learned_from_5_million_books_erez_lieberman_aiden_an  21.7M  17.1G  45.6K  5.07K 20130714 14 min 8 s
                 toy_tiles_that_talk_to_each_other_david_merrill  17.9M  17.2G  45.6K  5.07K 20130222 7 min 12 s
                                 our_buggy_moral_code_dan_ariely  39.5M  17.2G  45.5K  5.05K 20130203 16 min 51 s
                     mysteries_of_vernacular_noise_jessica_oreck  3.50M  17.2G  45.4K  5.04K 20130405 2 min 1 s
    the_shape_shifting_future_of_the_mobile_phone_fabian_hemmert  8.23M  17.2G  44.5K  4.94K 20130728 4 min 15 s
                  are_we_ready_for_neo_evolution_harvey_fineberg  28.5M  17.2G  44.4K  4.93K 20130324 17 min 21 s
                           the_post_crisis_consumer_john_gerzema  24.9M  17.3G  44.4K  4.93K 20130605 16 min 34 s
                              the_art_of_choosing_sheena_iyengar  40.0M  17.3G  43.9K  4.88K 20130824 24 min 8 s
                        beware_online_filter_bubbles_eli_pariser  16.3M  17.3G  43.4K  4.83K 20130322 9 min 4 s
                               why_videos_go_viral_kevin_allocca  13.9M  17.3G  43.4K  4.82K 20121117 7 min 20 s
               creative_houses_from_reclaimed_stuff_dan_phillips  33.1M  17.4G  43.0K  4.78K 20130717 17 min 57 s
              yup_i_built_a_nuclear_fusion_reactor_taylor_wilson  8.14M  17.4G  42.7K  4.74K 20130719 20 min 28 s
                     mysteries_of_vernacular_pants_jessica_oreck  3.38M  17.4G  42.7K  4.74K 20130402 2 min 2 s
                         moral_behavior_in_animals_frans_de_waal  36.4M  17.4G  42.4K  4.71K 20130703 16 min 52 s
                         pop_an_ollie_and_innovate_rodney_mullen  48.2M  17.5G  42.1K  4.68K 20130821 18 min 19 s
                    bring_ted_to_the_classroom_with_ted_ed_clubs  2.33M  17.5G  32.5K  4.64K 20150819 1 min 13 s
                          the_real_goal_equal_pay_for_equal_play  16.5M  17.5G  23.1K  4.61K 20170308 9 min 45 s
          making_a_ted_ed_lesson_two_ways_to_animate_slam_poetry  15.0M  17.5G  41.5K  4.61K 20130720 5 min 24 s
       toward_a_new_understanding_of_mental_illness_thomas_insel  22.1M  17.5G  41.5K  4.61K 20130707 13 min 6 s
                  shedding_light_on_dark_matter_patricia_burchat  26.7M  17.5G  41.4K  4.60K 20121130 17 min 8 s
                           redefining_the_dictionary_erin_mckean  29.9M  17.6G  41.2K  4.58K 20121228 15 min 54 s
                           a_new_way_to_diagnose_autism_ami_klin  37.7M  17.6G  40.6K  4.51K 20130602 19 min 44 s
                       planning_for_the_end_of_oil_richard_sears  20.1M  17.6G  40.3K  4.48K 20130301 6 min 52 s
                        seeing_a_sustainable_future_alex_steffen  15.3M  17.6G  40.3K  4.48K 20121207 10 min 13 s
    mysteries_of_vernacular_dynamite_jessica_oreck_and_rachael_t  3.53M  17.6G  40.0K  4.45K 20130607 2 min 15 s
                   sublimation_mit_digital_lab_techniques_manual  8.96M  17.7G  53.0K  4.42K 20100204 6 min 7 s
                       building_a_culture_of_success_mark_wilson  15.1M  17.7G  39.7K  4.41K 20130621 8 min 54 s
    mysteries_of_vernacular_x_ray_jessica_oreck_and_rachael_teel  2.58M  17.7G  39.6K  4.40K 20130805 1 min 58 s
                    mysteries_of_vernacular_tuxedo_jessica_oreck  3.59M  17.7G  39.5K  4.39K 20130501 2 min 4 s
              on_being_a_woman_and_a_diplomat_madeleine_albright  28.9M  17.7G  38.9K  4.32K 20130817 12 min 59 s
                the_surprising_science_of_happiness_nancy_etcoff  24.1M  17.7G  38.7K  4.30K 20130713 14 min 21 s
           the_search_for_other_earth_like_planets_olivier_guyon  13.7M  17.7G  38.6K  4.29K 20130425 6 min 20 s
                              the_3_a_s_of_awesome_neil_pasricha  41.3M  17.8G  38.6K  4.29K 20130825 17 min 33 s
       how_economic_inequality_harms_societies_richard_wilkinson  31.0M  17.8G  38.2K  4.24K 20130809 16 min 54 s
                 mysteries_of_vernacular_miniature_jessica_oreck  3.84M  17.8G  38.0K  4.22K 20130419 2 min 3 s
                                   on_being_wrong_kathryn_schulz  35.0M  17.8G  37.9K  4.21K 20130315 17 min 51 s
          medicine_s_future_there_s_an_app_for_that_daniel_kraft  25.1M  17.9G  37.2K  4.13K 20130623 18 min 21 s
        detention_or_eco_club_choosing_your_future_juan_martinez  9.35M  17.9G  37.0K  4.11K 20130104 6 min 38 s
                             erin_mckean_the_joy_of_lexicography  32.7M  17.9G  61.6K  4.11K 20070830 17 min 47 s
             fractals_and_the_art_of_roughness_benoit_mandelbrot  34.0M  17.9G  36.7K  4.07K 20130308 17 min 9 s
                       a_warm_embrace_that_saves_lives_jane_chen  8.86M  18.0G  36.5K  4.05K 20130810 4 min 46 s
                     how_to_learn_from_mistakes_diana_laufenberg  21.9M  18.0G  36.3K  4.03K 20130818 10 min 5 s
                        how_to_restore_a_rainforest_willie_smits  40.2M  18.0G  36.3K  4.03K 20130222 20 min 39 s
                       how_i_fell_in_love_with_a_fish_dan_barber  39.0M  18.1G  36.2K  4.02K 20130215 19 min 32 s
                        are_we_born_to_run_christopher_mcdougall  45.3M  18.1G  35.7K  3.97K 20130808 15 min 52 s
                                       announcing_ted_ed_espanol  2.06M  18.1G  27.8K  3.97K 20150415 34 s 387 ms
    mysteries_of_vernacular_window_jessica_oreck_and_rachael_tee  3.31M  18.1G  35.5K  3.94K 20130613 1 min 56 s
                the_lost_art_of_democratic_debate_michael_sandel  54.1M  18.2G  35.2K  3.91K 20130323 19 min 42 s
                         navigating_our_global_future_ian_goldin  12.2M  18.2G  35.1K  3.90K 20130817 7 min 6 s
    mysteries_of_vernacular_sarcophagus_jessica_oreck_and_rachae  2.57M  18.2G  35.0K  3.89K 20130726 1 min 37 s
    mysteries_of_vernacular_venom_jessica_oreck_and_rachael_teel  3.18M  18.2G  34.7K  3.86K 20130607 2 min 2 s
    mysteries_of_vernacular_keister_jessica_oreck_and_rachael_te  2.86M  18.2G  34.6K  3.85K 20130809 2 min 0 s
                              the_security_mirage_bruce_schneier  40.7M  18.2G  34.2K  3.80K 20130612 21 min 5 s
                     txtng_is_killing_language_jk_john_mcwhorter  27.1M  18.2G  34.1K  3.79K 20130830 13 min 51 s
                mysteries_of_vernacular_inaugurate_jessica_oreck  3.45M  18.2G  34.1K  3.79K 20130524 2 min 7 s
              how_will_ted_ed_celebrate_its_1000000th_subscriber  2.80M  18.3G  26.4K  3.77K 20150114 49 s 399 ms
                     the_other_inconvenient_truth_jonathan_foley  29.5M  18.3G  33.8K  3.76K 20130821 17 min 46 s
                                           redefining_the_f_word  13.2M  18.3G  15.0K  3.74K 20171207 7 min 57 s
            every_city_needs_healthy_honey_bees_noah_wilson_rich  28.1M  18.3G  33.6K  3.73K 20130414 12 min 42 s
                 the_mathematics_of_history_jean_baptiste_michel  7.24M  18.3G  33.5K  3.72K 20130504 4 min 26 s
                 the_coming_neurological_epidemic_gregory_petsko  5.35M  18.3G  33.2K  3.69K 20130127 3 min 50 s
                           my_seven_species_of_robot_dennis_hong  30.5M  18.4G  32.8K  3.65K 20130815 16 min 15 s
                    digging_for_humanity_s_origins_louise_leakey  31.1M  18.4G  32.8K  3.64K 20130120 15 min 33 s
                                       exciting_news_from_ted_ed  5.13M  18.4G  32.7K  3.63K 20130423 2 min 13 s
    mysteries_of_vernacular_fizzle_jessica_oreck_and_rachael_tee  2.65M  18.4G  32.7K  3.63K 20130719 1 min 50 s
               dare_to_educate_afghan_girls_shabana_basij_rasikh  21.9M  18.4G  32.5K  3.61K 20130421 9 min 36 s
                  building_the_seed_cathedral_thomas_heatherwick  31.9M  18.5G  32.2K  3.58K 20130330 16 min 52 s
     mysteries_of_vernacular_jade_jessica_oreck_and_rachael_teel  3.39M  18.5G  32.2K  3.57K 20130712 2 min 7 s
                 a_rosetta_stone_for_the_indus_script_rajesh_rao  27.9M  18.5G  31.8K  3.53K 20130405 17 min 1 s
                   hey_science_teachers_make_it_fun_tyler_dewitt  28.3M  18.5G  31.8K  3.53K 20130422 14 min 10 s
                      why_eyewitnesses_get_it_wrong_scott_fraser  43.1M  18.6G  31.6K  3.51K 20130703 18 min 26 s
                  ted_prize_wish_protect_our_oceans_sylvia_earle  55.7M  18.6G  31.5K  3.49K 20121221 18 min 11 s
                   faith_versus_tradition_in_islam_mustafa_akyol  27.2M  18.6G  31.3K  3.48K 20130808 17 min 11 s
                      symmetry_reality_s_riddle_marcus_du_sautoy  40.1M  18.7G  31.2K  3.47K 20130817 18 min 19 s
                              how_to_use_a_paper_towel_joe_smith  11.2M  18.7G  31.1K  3.46K 20130821 4 min 31 s
                               your_brain_on_improv_charles_limb  29.6M  18.7G  30.2K  3.36K 20130703 16 min 31 s
           the_quest_to_understand_consciousness_antonio_damasio  31.8M  18.7G  30.2K  3.35K 20130412 18 min 42 s
                    mysteries_of_vernacular_hearse_jessica_oreck  3.73M  18.7G  30.0K  3.34K 20130404 2 min 13 s
                                            meet_julia_delmedico  5.45M  18.8G  30.0K  3.33K 20130508 2 min 16 s
                       feats_of_memory_anyone_can_do_joshua_foer  33.5M  18.8G  29.9K  3.33K 20130426 20 min 28 s
               building_a_museum_of_museums_on_the_web_amit_sood  9.86M  18.8G  29.7K  3.30K 20130322 5 min 35 s
    why_domestic_violence_victims_don_t_leave_leslie_morgan_stei  33.8M  18.8G  28.9K  3.21K 20130520 15 min 59 s
                  high_tech_art_with_a_sense_of_humor_aparna_rao  12.7M  18.8G  28.6K  3.18K 20121202 8 min 20 s
                 404_the_story_of_a_page_not_found_renny_gleeson  7.71M  18.8G  28.4K  3.16K 20130426 4 min 7 s
               healthier_men_one_moustache_at_a_time_adam_garone  47.6M  18.9G  27.6K  3.07K 20130407 16 min 41 s
                 behind_the_great_firewall_of_china_michael_anti  35.3M  18.9G  27.6K  3.06K 20130622 18 min 51 s
                   the_beautiful_math_of_coral_margaret_wertheim  28.2M  19.0G  27.2K  3.02K 20121124 15 min 31 s
              will_our_kids_be_a_different_species_juan_enriquez  27.3M  19.0G  26.9K  2.99K 20130821 16 min 48 s
                              the_demise_of_guys_philip_zimbardo  8.43M  19.0G  26.5K  2.94K 20121216 4 min 46 s
                                 the_art_of_asking_amanda_palmer  29.5M  19.0G  26.3K  2.92K 20130830 13 min 47 s
               the_business_logic_of_sustainability_ray_anderson  33.2M  19.1G  26.3K  2.92K 20130301 15 min 51 s
                  building_a_dinosaur_from_a_chicken_jack_horner  25.4M  19.1G  26.1K  2.90K 20130329 16 min 36 s
               what_you_don_t_know_about_marriage_jenna_mccarthy  18.1M  19.1G  25.8K  2.87K 20130815 11 min 17 s
                       making_a_ted_ed_lesson_concept_and_design  9.81M  19.1G  25.7K  2.85K 20130527 3 min 16 s
                        the_good_news_of_the_decade_hans_rosling  29.8M  19.1G  25.6K  2.85K 20130811 15 min 34 s
           imaging_at_a_trillion_frames_per_second_ramesh_raskar  20.9M  19.2G  25.6K  2.85K 20130621 11 min 1 s
                        the_myth_of_the_gay_agenda_lz_granderson  44.1M  19.2G  25.3K  2.82K 20130821 17 min 51 s
       your_brain_is_more_than_a_bag_of_chemicals_david_anderson  29.0M  19.2G  25.3K  2.81K 20130512 15 min 25 s
                                              meet_melissa_perez  5.01M  19.2G  25.2K  2.80K 20130508 2 min 33 s
            4_lessons_from_robots_about_being_human_ken_goldberg  29.5M  19.3G  23.9K  2.66K 20130707 17 min 9 s
                      why_we_need_to_go_back_to_mars_joel_levine  27.5M  19.3G  23.8K  2.65K 20130801 16 min 14 s
                                do_the_green_thing_andy_hobsbawm  6.83M  19.3G  23.7K  2.64K 20130126 3 min 55 s
              the_el_sistema_music_revolution_jose_antonio_abreu  36.7M  19.3G  23.7K  2.63K 20130222 17 min 24 s
                  ladies_and_gentlemen_the_hobart_shakespeareans  28.0M  19.4G  23.6K  2.62K 20130516 10 min 9 s
                            ted_ed_clubs_presents_ted_ed_weekend  8.88M  19.4G  13.0K  2.60K 20170223 3 min 32 s
                    superhero_training_what_you_can_do_right_now  31.8M  19.4G  10.4K  2.59K 20180301 10 min 20 s
            music_and_emotion_through_time_michael_tilson_thomas  38.8M  19.4G  23.3K  2.58K 20130426 20 min 13 s
                my_backyard_got_way_cooler_when_i_added_a_dragon  13.3M  19.4G  10.3K  2.56K 20171108 7 min 15 s
                                 how_to_spot_a_liar_pamela_meyer  31.3M  19.5G  22.9K  2.54K 20130809 18 min 50 s
                             the_walk_from_no_to_yes_william_ury  36.8M  19.5G  22.9K  2.54K 20130818 18 min 45 s
                                      doodlers_unite_sunni_brown  10.0M  19.5G  22.9K  2.54K 20130405 5 min 50 s
                        retrofitting_suburbia_ellen_dunham_jones  35.5M  19.6G  22.8K  2.54K 20121214 19 min 24 s
                                                meet_shayna_cody  3.82M  19.6G  22.5K  2.50K 20130508 1 min 36 s
                                 losing_everything_david_hoffman  11.2M  19.6G  22.4K  2.49K 20130119 5 min 6 s
             the_hidden_beauty_of_pollination_louie_schwartzberg  24.1M  19.6G  22.2K  2.46K 20130322 7 min 40 s
                     how_poachers_became_caretakers_john_kasaona  27.9M  19.6G  17.0K  2.43K 20150210 15 min 46 s
                           the_equation_for_reaching_your_dreams  25.3M  19.6G  9.65K  2.41K 20180508 9 min 22 s
                              archeology_from_space_sarah_parcak  9.71M  19.7G  21.7K  2.41K 20130507 5 min 20 s
      facing_the_real_me_looking_in_the_mirror_with_natural_hair  28.9M  19.7G  9.49K  2.37K 20180209 11 min 25 s
                                           be_you_ty_over_beauty  21.9M  19.7G  11.8K  2.37K 20170823 10 min 25 s
             let_s_raise_kids_to_be_entrepreneurs_cameron_herold  34.9M  19.7G  21.2K  2.36K 20130801 19 min 36 s
                               how_a_fly_flies_michael_dickinson  26.6M  19.8G  21.1K  2.34K 20130508 15 min 55 s
        making_sense_of_a_visible_quantum_object_aaron_o_connell  15.7M  19.8G  21.0K  2.33K 20130329 7 min 51 s
                        the_sound_the_universe_makes_janna_levin  36.3M  19.8G  20.9K  2.32K 20130315 17 min 43 s
                            a_light_switch_for_neurons_ed_boyden  42.0M  19.9G  20.8K  2.31K 20130406 18 min 24 s
                              what_do_babies_think_alison_gopnik  38.1M  19.9G  20.6K  2.28K 20130809 18 min 29 s
    how_mr_condom_made_thailand_a_better_place_mechai_viravaidya  24.8M  19.9G  20.5K  2.28K 20130710 13 min 50 s
                                   praising_slowness_carl_honore  60.5M  20.0G  20.3K  2.26K 20130726 19 min 17 s
                      a_future_beyond_traffic_gridlock_bill_ford  29.2M  20.0G  20.3K  2.26K 20130413 16 min 48 s
                    overcoming_the_scientific_divide_aaron_reedy  13.3M  20.0G  19.9K  2.21K 20130620 7 min 56 s
    image_recognition_that_triggers_augmented_reality_matt_mills  21.2M  20.0G  19.9K  2.21K 20130629 8 min 4 s
                     how_benjamin_button_got_his_face_ed_ulbrich  38.8M  20.1G  19.6K  2.18K 20130222 18 min 4 s
                          teachers_need_real_feedback_bill_gates  24.7M  20.1G  19.6K  2.17K 20130816 10 min 24 s
                   could_a_saturn_moon_harbor_life_carolyn_porco  7.68M  20.1G  19.5K  2.17K 20130301 3 min 26 s
                                      social_animal_david_brooks  35.9M  20.1G  19.3K  2.15K 20130317 18 min 43 s
                   what_s_so_funny_about_mental_illness_ruby_wax  27.3M  20.2G  19.3K  2.15K 20130524 8 min 45 s
                   a_new_ecosystem_for_electric_cars_shai_agassi  36.8M  20.2G  19.2K  2.14K 20130301 18 min 3 s
                      tour_the_solar_system_from_home_jon_nguyen  12.3M  20.2G  19.2K  2.13K 20130821 7 min 53 s
                 four_principles_for_the_open_world_don_tapscott  42.3M  20.3G  19.0K  2.11K 20130208 17 min 50 s
                       supercharged_motorcycle_design_yves_behar  4.62M  20.3G  18.6K  2.07K 20130301 2 min 20 s
                  the_beautiful_tricks_of_flowers_jonathan_drori  21.6M  20.3G  18.5K  2.05K 20130831 13 min 48 s
                artificial_justice_would_robots_make_good_judges  14.1M  20.3G  8.14K  2.04K 20180320 6 min 37 s
                               battling_bad_science_ben_goldacre  30.4M  20.3G  18.0K  2.00K 20130809 14 min 19 s
              a_tale_of_mental_illness_from_the_inside_elyn_saks  34.2M  20.4G  17.9K  1.99K 20130628 14 min 52 s
             lessons_from_fashion_s_free_culture_johanna_blakley  20.7M  20.4G  17.5K  1.94K 20130801 15 min 36 s
    a_navy_admiral_s_thoughts_on_global_security_james_stavridis  37.4M  20.4G  17.3K  1.92K 20130621 16 min 34 s
          from_mach_20_glider_to_humming_bird_drone_regina_dugan  44.8M  20.5G  17.3K  1.92K 20130420 25 min 1 s
    cheese_dogs_and_a_pill_to_kill_mosquitoes_and_end_malaria_ba  22.9M  20.5G  17.2K  1.91K 20130602 10 min 20 s
                                  the_earth_is_full_paul_gilding  26.8M  20.5G  17.2K  1.91K 20130412 16 min 46 s
                         making_a_ted_ed_lesson_creative_process  10.3M  20.5G  17.1K  1.90K 20130527 4 min 14 s
                natural_pest_control_using_bugs_shimon_steinberg  29.6M  20.6G  15.1K  1.88K 20130913 15 min 23 s
                             dare_to_disagree_margaret_heffernan  21.2M  20.6G  16.8K  1.86K 20130614 8 min 44 s
                           unintended_consequences_edward_tenner  24.9M  20.6G  16.6K  1.85K 20130405 16 min 10 s
                a_saudi_woman_who_dared_to_drive_manal_al_sharif  33.2M  20.6G  16.5K  1.83K 20130823 14 min 19 s
    why_libya_s_revolution_didn_t_work_and_what_might_zahra_lang  28.5M  20.7G  16.5K  1.83K 20130612 9 min 47 s
          distant_time_and_the_hint_of_a_multiverse_sean_carroll  26.5M  20.7G  16.4K  1.82K 20130703 15 min 54 s
                                     atheism_2_0_alain_de_botton  46.6M  20.7G  16.4K  1.82K 20130726 19 min 20 s
                      making_a_car_for_blind_drivers_dennis_hong  20.6M  20.7G  16.4K  1.82K 20130329 9 min 8 s
                                   tribal_leadership_david_logan  34.7M  20.8G  16.0K  1.78K 20130717 16 min 39 s
                          why_global_jihad_is_losing_bobby_ghosh  30.8M  20.8G  16.0K  1.78K 20130619 16 min 31 s
                 how_your_mom_s_advice_could_save_the_human_race  23.3M  20.8G  7.11K  1.78K 20180329 9 min 22 s
               the_true_power_of_the_performing_arts_ben_cameron  27.2M  20.9G  15.7K  1.75K 20130906 12 min 44 s
             the_rise_of_human_computer_cooperation_shyam_sankar  20.1M  20.9G  15.6K  1.74K 20130615 12 min 12 s
                 we_need_a_moral_operating_system_damon_horowitz  43.7M  20.9G  15.6K  1.73K 20130808 16 min 18 s
                         let_s_simplify_legal_jargon_alan_siegel  9.12M  20.9G  15.4K  1.72K 20130218 4 min 57 s
             a_cinematic_journey_through_visual_effects_don_levy  20.5M  21.0G  15.3K  1.70K 20130519 6 min 54 s
            your_online_life_permanent_as_a_tattoo_juan_enriquez  8.77M  21.0G  15.3K  1.70K 20130830 6 min 0 s
              the_game_layer_on_top_of_the_world_seth_priebatsch  22.2M  21.0G  15.2K  1.69K 20130906 12 min 22 s
    what_doctors_don_t_know_about_the_drugs_they_prescribe_ben_g  29.1M  21.0G  15.1K  1.68K 20130531 13 min 29 s
                                             meet_shahruz_ghaemi  4.58M  21.0G  15.1K  1.67K 20130514 1 min 58 s
            women_should_represent_women_in_media_megan_kamerick  21.3M  21.0G  15.0K  1.67K 20130821 10 min 31 s
                              why_democracy_matters_rory_stewart  24.9M  21.1G  15.0K  1.67K 20130605 13 min 41 s
                       one_way_to_create_a_more_inclusive_school  16.7M  21.1G  8.24K  1.65K 20170726 8 min 18 s
         the_moral_dangers_of_non_lethal_weapons_stephen_coleman  52.0M  21.1G  14.8K  1.64K 20130714 17 min 32 s
               the_science_behind_a_climate_headline_rachel_pike  7.32M  21.1G  14.7K  1.64K 20130809 4 min 13 s
             protecting_the_brain_against_concussion_kim_gorgens  18.6M  21.2G  14.5K  1.61K 20130816 9 min 21 s
       how_youtube_thinks_about_copyright_margaret_gould_stewart  11.6M  21.2G  14.0K  1.55K 20130308 5 min 46 s
                     the_emergence_of_4d_printing_skylar_tibbits  15.8M  21.2G  13.8K  1.54K 20130823 8 min 26 s
          how_arduino_is_open_sourcing_imagination_massimo_banzi  28.4M  21.2G  13.3K  1.47K 20130705 15 min 46 s
          the_beautiful_nano_details_of_our_world_gary_greenberg  18.9M  21.2G  11.8K  1.47K 20131211 12 min 6 s
                how_to_use_experts_and_when_not_to_noreena_hertz  37.2M  21.3G  13.2K  1.46K 20130824 18 min 18 s
                               religions_and_babies_hans_rosling  22.2M  21.3G  13.2K  1.46K 20130821 13 min 20 s
                                 the_evolution_of_the_human_head  13.8M  21.3G  16.0K  1.45K 20110131 5 min 3 s
                   the_generation_that_s_remaking_china_yang_lan  33.6M  21.3G  13.1K  1.45K 20130809 17 min 14 s
                            trust_morality_and_oxytocin_paul_zak  37.0M  21.4G  12.8K  1.42K 20130809 16 min 34 s
              how_i_m_preparing_to_get_alzheimer_s_alanna_shaikh  12.8M  21.4G  12.2K  1.36K 20130628 20 min 28 s
                        how_to_solve_traffic_jams_jonas_eliasson  16.4M  21.4G  12.1K  1.34K 20130415 8 min 27 s
                       inside_an_antarctic_time_machine_lee_hotz  14.9M  21.4G  11.9K  1.32K 20130810 9 min 45 s
                 using_unanswered_questions_to_teach_john_gensic  8.99M  21.4G  11.8K  1.31K 20130620 5 min 24 s
                                    a_future_lit_by_solar_energy  34.4M  21.5G  5.21K  1.30K 20180202 13 min 0 s
                a_global_culture_to_fight_extremism_maajid_nawaz  43.1M  21.5G  11.3K  1.26K 20121215 17 min 53 s
                    a_radical_experiment_in_empathy_sam_richards  31.9M  21.5G  11.2K  1.24K 20130815 18 min 7 s
         want_to_help_someone_shut_up_and_listen_ernesto_sirolli  36.4M  21.6G  11.2K  1.24K 20130619 17 min 9 s
                 the_journey_across_the_high_wire_philippe_petit  32.7M  21.6G  11.2K  1.24K 20130507 19 min 7 s
                               greening_the_ghetto_majora_carter  50.5M  21.6G  11.1K  1.23K 20130727 18 min 33 s
                toward_a_science_of_simplicity_george_whitesides  34.4M  21.7G  11.0K  1.22K 20130222 19 min 5 s
                      gaming_for_understanding_brenda_brathwaite  15.4M  21.7G  11.0K  1.22K 20130821 9 min 23 s
          how_can_technology_transform_the_human_body_lucy_mcrae  8.98M  21.7G  11.0K  1.22K 20130419 3 min 59 s
                  agile_programming_for_your_family_bruce_feiler  38.5M  21.7G  9.70K  1.21K 20131203 18 min 0 s
                        are_droids_taking_our_jobs_andrew_mcafee  28.5M  21.8G  10.8K  1.20K 20130829 14 min 7 s
                       the_real_reason_for_brains_daniel_wolpert  34.9M  21.8G  10.6K  1.18K 20130809 19 min 59 s
                             can_we_domesticate_germs_paul_ewald  29.9M  21.8G  10.5K  1.17K 20130115 17 min 49 s
                                   the_optimism_bias_tali_sharot  30.9M  21.9G  10.4K  1.16K 20130426 17 min 40 s
      the_key_to_growth_race_with_the_machines_erik_brynjolfsson  19.7M  21.9G  10.4K  1.16K 20130830 11 min 59 s
            our_failing_schools_enough_is_enough_geoffrey_canada  50.1M  21.9G  10.3K  1.14K 20130816 17 min 10 s
        telling_my_whole_story_when_many_cultures_make_one_voice  38.6M  22.0G  4.48K  1.12K 20171121 12 min 54 s
                animating_a_photo_real_digital_face_paul_debevec  10.2M  22.0G  10.1K  1.12K 20130717 6 min 6 s
                               click_your_fortune_episode_2_demo  4.97M  22.0G  9.98K  1.11K 20130807 3 min 25 s
                                  a_map_of_the_brain_allan_jones  35.0M  22.0G  9.91K  1.10K 20130809 15 min 21 s
                    learning_from_a_barefoot_movement_bunker_roy  37.1M  22.1G  9.80K  1.09K 20130809 19 min 7 s
                       ancient_wonders_captured_in_3d_ben_kacyra  29.8M  22.1G  9.46K  1.05K 20130809 12 min 20 s
    a_critical_examination_of_the_technology_in_our_lives_kevin_  12.7M  22.1G  9.37K  1.04K 20130625 7 min 19 s
                         reinventing_the_encyclopedia_game_rives  24.2M  22.1G  9.33K  1.04K 20130828 10 min 45 s
      building_a_theater_that_remakes_itself_joshua_prince_ramus  30.5M  22.1G  9.13K  1.01K 20130717 18 min 42 s
               sex_needs_a_new_metaphor_here_s_one_al_vernacchio  13.8M  22.2G  9.06K  1.01K 20130816 8 min 24 s
                     how_we_ll_stop_polio_for_good_bruce_aylward  38.7M  22.2G  8.96K  1.00K 20130329 23 min 9 s
                                   hire_the_hackers_misha_glenny  41.5M  22.2G  8.94K  0.99K 20130816 18 min 39 s
                       perspective_is_everything_rory_sutherland  36.0M  22.3G  8.83K  0.98K 20130821 18 min 24 s
                       the_secret_of_the_bat_genome_emma_teeling  32.3M  22.3G  8.65K    984 20130605 16 min 25 s
          meet_global_corruption_s_hidden_players_charmian_gooch  31.3M  22.3G  8.50K    967 20130823 14 min 30 s
                         what_is_the_internet_really_andrew_blum  17.1M  22.4G  8.48K    965 20130531 8 min 44 s
                            a_giant_bubble_for_debate_liz_diller  18.5M  22.4G  8.45K    961 20130419 12 min 6 s
                       the_4_commandments_of_cities_eduardo_paes  22.6M  22.4G  8.42K    958 20130712 12 min 21 s
                    what_fear_can_teach_us_karen_thompson_walker  25.4M  22.4G  8.16K    928 20130517 11 min 30 s
                        we_need_better_drugs_now_francis_collins  25.9M  22.4G  8.16K    928 20130830 14 min 40 s
                         trusting_the_ensemble_charles_hazlewood  52.0M  22.5G  8.15K    927 20130809 19 min 36 s
         what_we_re_learning_from_online_education_daphne_koller  40.9M  22.5G  7.93K    902 20130614 20 min 40 s
              what_we_re_learning_from_5000_brains_read_montague  26.2M  22.6G  7.91K    899 20130531 13 min 23 s
        weaving_narratives_in_museum_galleries_thomas_p_campbell  34.8M  22.6G  7.91K    899 20130511 16 min 36 s
                      mining_minerals_from_seawater_damian_palin  4.76M  22.6G  7.80K    887 20130705 3 min 1 s
              how_poachers_became_caretakers_john_kasaona_glitch  28.4M  22.6G  7.65K    869 20130310 15 min 46 s
                            shape_shifting_dinosaurs_jack_horner  36.1M  22.7G  7.36K    837 20130815 18 min 22 s
                           texting_that_saves_lives_nancy_lublin  12.3M  22.7G  7.31K    832 20130712 5 min 24 s
                               click_your_fortune_episode_3_demo  5.18M  22.7G  7.29K    829 20130807 3 min 31 s
                         why_i_hold_on_to_my_grandmother_s_tales  13.0M  22.7G  3.21K    821 20180117 7 min 8 s
              obesity_hunger_1_global_food_issue_ellen_gustafson  15.4M  22.7G  7.16K    815 20130801 16 min 34 s
             a_broken_body_isn_t_a_broken_person_janine_shepherd  34.8M  22.7G  7.08K    805 20130528 18 min 57 s
           the_case_for_collaborative_consumption_rachel_botsman  29.1M  22.8G  6.97K    793 20130717 16 min 34 s
                   how_to_defend_earth_from_asteroids_phil_plait  22.7M  22.8G  6.95K    790 20130808 14 min 16 s
              ultrasound_surgery_healing_without_cuts_yoav_medan  30.7M  22.8G  6.95K    790 20130719 16 min 13 s
                 a_whistleblower_you_haven_t_heard_geert_chatrou  21.9M  22.8G  6.92K    787 20130815 11 min 56 s
                          let_s_teach_kids_to_code_mitch_resnick  29.1M  22.9G  6.92K    787 20130424 16 min 48 s
                               a_doctor_s_touch_abraham_verghese  38.3M  22.9G  6.89K    783 20130816 18 min 32 s
                       how_we_found_the_giant_squid_edith_widder  15.2M  22.9G  6.78K    771 20130830 8 min 38 s
                      a_plane_you_can_drive_anna_mracek_dietrich  19.5M  22.9G  6.75K    767 20130809 9 min 37 s
                   a_civil_response_to_violence_emiliano_salinas  20.4M  23.0G  6.67K    759 20130808 12 min 17 s
         let_s_transform_energy_with_natural_gas_t_boone_pickens  34.6M  23.0G  6.65K    756 20130719 19 min 42 s
                          re_examining_the_remix_lawrence_lessig  28.9M  23.0G  6.65K    756 20130801 18 min 45 s
                             how_to_buy_happiness_michael_norton  17.8M  23.0G  6.63K    754 20130821 10 min 58 s
                    pool_medical_patents_save_lives_ellen_t_hoen  18.6M  23.1G  6.63K    754 20130428 11 min 16 s
                  the_flavors_of_life_through_the_eyes_of_a_chef  17.4M  23.1G  2.91K    744 20171130 6 min 39 s
    what_nonprofits_can_learn_from_coca_cola_melinda_french_gate  33.8M  23.1G  6.50K    739 20130815 16 min 28 s
                                kids_need_structure_colin_powell  33.4M  23.1G  6.49K    738 20130401 17 min 46 s
               a_mini_robot_powered_by_your_phone_keller_rinaudo  12.0M  23.2G  6.42K    730 20130823 5 min 54 s
                         the_right_time_to_second_guess_yourself  22.4M  23.2G  3.56K    728 20170810 11 min 56 s
                             meet_the_water_canary_sonaar_luthra  5.73M  23.2G  6.40K    728 20130726 3 min 37 s
                         how_you_can_help_fight_pediatric_cancer  14.4M  23.2G  3.52K    721 20170608 8 min 6 s
         a_universal_translator_for_surgeons_steven_schwaitzberg  19.3M  23.2G  6.31K    718 20130519 11 min 41 s
           the_future_race_car_150mph_and_no_driver_chris_gerdes  29.8M  23.2G  6.31K    717 20130829 10 min 47 s
                   a_vision_of_crimes_in_the_future_marc_goodman  20.6M  23.3G  6.26K    712 20130628 8 min 44 s
     how_do_you_save_a_shark_you_know_nothing_about_simon_berrow  35.7M  23.3G  6.17K    702 20130626 16 min 46 s
        let_s_put_birth_control_back_on_the_agenda_melinda_gates  55.2M  23.3G  6.09K    693 20130605 25 min 36 s
                 a_father_daughter_dance_in_prison_angela_patton  22.3M  23.4G  5.37K    687 20131211 8 min 48 s
             the_good_news_on_poverty_yes_there_s_good_news_bono  24.9M  23.4G  5.97K    678 20130823 13 min 57 s
               what_we_learn_before_we_re_born_annie_murphy_paul  21.3M  23.4G  5.96K    678 20130809 8 min 44 s
      sometimes_it_s_good_to_give_up_the_driver_s_seat_baba_shiv  24.8M  23.4G  5.89K    670 20130501 9 min 47 s
    unlock_the_intelligence_passion_greatness_of_girls_leymah_gb  44.7M  23.5G  5.88K    669 20130712 20 min 28 s
                         a_prosthetic_arm_that_feels_todd_kuiken  38.2M  23.5G  5.76K    655 20130809 18 min 51 s
                        1000_tedtalks_6_words_sebastian_wernicke  13.3M  23.5G  5.76K    655 20130815 7 min 33 s
                      my_friend_richard_feynman_leonard_susskind  27.6M  23.6G  5.66K    644 20130815 14 min 41 s
                                a_child_of_the_state_lemn_sissay  29.0M  23.6G  4.99K    638 20131211 15 min 36 s
            experiments_that_hint_of_longer_lives_cynthia_kenyon  31.1M  23.6G  5.51K    627 20130809 16 min 23 s
                   using_nature_to_grow_batteries_angela_belcher  19.7M  23.6G  5.52K    627 20130815 10 min 25 s
             could_the_sun_be_good_for_your_heart_richard_weller  23.7M  23.7G  5.46K    621 20130325 12 min 59 s
                  energy_from_floating_algae_pods_jonathan_trent  30.6M  23.7G  5.41K    615 20130607 14 min 45 s
                              before_i_die_i_want_to_candy_chang  11.3M  23.7G  5.37K    611 20130614 6 min 19 s
                          crowdsource_your_health_lucien_engelen  14.6M  23.7G  5.36K    609 20130815 6 min 12 s
                                         where_is_home_pico_iyer  31.2M  23.7G  5.31K    604 20130823 14 min 4 s
                       the_dance_of_the_dung_beetle_marcus_byrne  35.9M  23.8G  5.28K    600 20130429 17 min 8 s
                      lessons_from_death_row_inmates_david_r_dow  32.2M  23.8G  5.19K    590 20130821 18 min 16 s
             mental_health_for_all_by_involving_all_vikram_patel  32.5M  23.8G  5.12K    582 20130607 12 min 22 s
                  how_healthy_living_nearly_killed_me_a_j_jacobs  14.8M  23.9G  5.05K    574 20130726 8 min 42 s
                            how_do_we_heal_medicine_atul_gawande  33.9M  23.9G  5.03K    571 20130419 19 min 19 s
    what_s_a_snollygoster_a_short_lesson_in_political_speak_mark  15.6M  23.9G  5.00K    568 20130410 6 min 39 s
    science_is_for_everyone_kids_included_beau_lotto_and_amy_o_t  37.9M  23.9G  4.91K    558 20130524 15 min 25 s
                          how_to_topple_a_dictator_srdja_popovic  37.9M  24.0G  4.87K    553 20130815 12 min 3 s
                  let_s_prepare_for_our_new_climate_vicki_arroyo  21.3M  24.0G  4.86K    553 20130524 10 min 38 s
    parkinson_s_depression_and_the_switch_that_might_turn_them_o  24.7M  24.0G  4.68K    532 20130612 15 min 34 s
          gaming_to_re_engage_boys_in_learning_ali_carr_chellman  24.5M  24.1G  4.60K    523 20130815 12 min 30 s
                  neuroscience_game_theory_monkeys_colin_camerer  21.7M  24.1G  4.57K    520 20130619 13 min 49 s
                    the_voice_of_the_natural_world_bernie_krause  34.1M  24.1G  4.49K    510 20130816 14 min 51 s
    a_choreographer_s_creative_process_in_real_time_wayne_mcgreg  43.7M  24.1G  4.46K    507 20130607 15 min 18 s
    the_single_biggest_health_threat_women_face_noel_bairey_merz  31.5M  24.2G  4.46K    506 20130506 15 min 59 s
    we_the_people_and_the_republic_we_must_reclaim_lawrence_less  34.1M  24.2G  4.40K    500 20130823 18 min 22 s
                           how_to_expose_the_corrupt_peter_eigen  26.9M  24.2G  4.39K    499 20130725 16 min 12 s
                      inside_the_egyptian_revolution_wael_ghonim  26.8M  24.3G  4.38K    498 20130725 10 min 7 s
            michael_green_why_we_should_build_wooden_skyscrapers  23.7M  24.3G  4.38K    498 20130816 12 min 25 s
       a_clean_energy_proposal_race_to_the_top_jennifer_granholm  27.2M  24.3G  4.37K    497 20130830 12 min 41 s
         how_i_taught_rats_to_sniff_out_land_mines_bart_weetjens  22.5M  24.3G  4.26K    485 20130801 12 min 11 s
                 the_shared_experience_of_absurdity_charlie_todd  32.2M  24.4G  4.24K    482 20130808 12 min 4 s
    could_tissue_engineering_mean_personalized_medicine_nina_tan  10.3M  24.4G  4.20K    478 20130517 6 min 19 s
                                     my_immigration_story_tan_le  26.7M  24.4G  4.08K    464 20130815 12 min 16 s
                       older_people_are_happier_laura_carstensen  20.9M  24.4G  4.06K    462 20130821 11 min 38 s
                              we_can_recycle_plastic_mike_biddle  22.4M  24.4G  3.97K    452 20130809 10 min 58 s
                              print_your_own_medicine_lee_cronin  5.44M  24.5G  3.94K    447 20130510 3 min 6 s
                    how_to_look_inside_the_brain_carl_schoonover  9.04M  24.5G  3.89K    442 20130712 4 min 59 s
                  building_unimaginable_shapes_michael_hansmeyer  24.9M  24.5G  3.84K    436 20130621 11 min 7 s
    how_to_step_up_in_the_face_of_disaster_caitria_morgan_o_neil  19.7M  24.5G  3.77K    428 20130829 9 min 23 s
            a_prosthetic_eye_to_treat_blindness_sheila_nirenberg  19.1M  24.5G  3.66K    416 20130726 10 min 5 s
                                embrace_the_remix_kirby_ferguson  17.5M  24.5G  3.53K    401 20130510 9 min 42 s
                           advice_to_young_scientists_e_o_wilson  27.0M  24.6G  3.45K    392 20130705 14 min 56 s
                   the_strange_politics_of_disgust_david_pizarro  27.5M  24.6G  3.38K    384 20130408 14 min 2 s
                             what_s_left_to_explore_nathan_wolfe  14.1M  24.6G  3.32K    377 20130507 7 min 10 s
      what_if_our_healthcare_system_kept_us_healthy_rebecca_onie  29.6M  24.6G  3.31K    376 20130705 16 min 34 s
          visualizing_the_medical_data_explosion_anders_ynnerman  25.7M  24.7G  3.27K    371 20130710 16 min 34 s
                       the_100000_student_classroom_peter_norvig  11.4M  24.7G  3.23K    367 20130705 6 min 11 s
         a_teacher_growing_green_in_the_south_bronx_stephen_ritz  31.5M  24.7G  3.22K    366 20130829 13 min 42 s
                               tracking_the_trackers_gary_kovacs  11.2M  24.7G  3.21K    364 20130426 6 min 39 s
                      fighting_with_non_violence_scilla_elworthy  13.8M  24.7G  3.19K    363 20130619 8 min 44 s
           why_architects_need_to_use_their_ears_julian_treasure  16.5M  24.7G  3.11K    353 20130531 9 min 51 s
                 the_secret_lives_of_paintings_maurizio_seracini  21.8M  24.8G  2.91K    331 20130524 12 min 34 s
                            what_s_your_200_year_plan_raghava_kk  20.8M  24.8G  2.92K    331 20130829 10 min 58 s
                                     life_s_third_act_jane_fonda  26.0M  24.8G  2.86K    325 20130815 11 min 20 s
                             be_an_artist_right_now_young_ha_kim  38.2M  24.8G  2.85K    324 20130612 16 min 57 s
                           making_sense_of_maps_aris_venetikidis  36.0M  24.9G  2.82K    321 20130529 16 min 35 s
                fighting_a_contagious_cancer_elizabeth_murchison  26.7M  24.9G  2.77K    315 20130816 13 min 3 s
       tracking_ancient_diseases_using_plaque_christina_warinner  9.05M  24.9G  2.76K    313 20130712 5 min 31 s
    experiments_that_point_to_a_new_understanding_of_cancer_mina  36.7M  25.0G  2.74K    311 20130628 16 min 17 s
                 cloudy_with_a_chance_of_joy_gavin_pretor_pinney  18.9M  25.0G  2.73K    310 20130816 10 min 57 s
                   design_for_people_not_awards_timothy_prestero  20.8M  25.0G  2.72K    309 20130829 11 min 4 s
                              the_art_of_creating_awe_rob_legato  50.3M  25.0G  2.56K    291 20130614 16 min 27 s
                  can_democracy_exist_without_trust_ivan_krastev  27.9M  25.1G  2.53K    288 20130510 14 min 4 s
           the_promise_of_research_with_stem_cells_susan_solomon  24.4M  25.1G  2.43K    276 20130607 15 min 31 s
                            a_census_of_the_ocean_paul_snelgrove  27.5M  25.1G  2.39K    271 20130719 16 min 47 s
                   finding_life_we_can_t_imagine_christoph_adami  41.4M  25.2G  2.38K    270 20130808 18 min 51 s
                      your_phone_company_is_watching_malte_spitz  27.3M  25.2G  2.32K    264 20130621 10 min 10 s
               welcome_to_the_genomic_revolution_richard_resnick  20.1M  25.2G  2.32K    263 20130703 11 min 2 s
                             technology_s_epic_story_kevin_kelly  31.7M  25.2G  2.28K    259 20130725 16 min 32 s
                freeing_energy_from_the_grid_justin_hall_tipping  30.8M  25.3G  2.21K    251 20130809 12 min 45 s
                       excuse_me_may_i_rent_your_car_robin_chase  31.3M  25.3G  2.14K    243 20130517 12 min 24 s
                     the_economic_injustice_of_plastic_van_jones  41.7M  25.3G  2.10K    238 20130808 17 min 33 s
      how_open_data_is_changing_international_aid_sanjay_pradhan  31.8M  25.4G  2.09K    237 20130517 15 min 20 s
                       a_girl_who_demanded_school_kakenya_ntaiya  44.2M  25.4G  2.08K    236 20130513 15 min 16 s
                      demand_a_fair_trade_cell_phone_bandi_mbubi  15.2M  25.4G  2.03K    230 20130829 9 min 21 s
             what_your_designs_say_about_you_sebastian_deterding  18.0M  25.4G  1.95K    221 20130821 12 min 20 s
                demand_a_more_open_source_government_beth_noveck  41.4M  25.5G  1.92K    217 20130607 17 min 23 s
                     a_story_about_knots_and_surgeons_ed_gavagan  24.7M  25.5G  1.85K    210 20130531 12 min 21 s
                       let_s_pool_our_medical_data_john_wilbanks  32.1M  25.5G  1.75K    198 20130524 16 min 25 s
         ethical_riddles_in_hiv_research_boghuma_kabisen_titanji  20.8M  25.6G  1.65K    187 20130605 11 min 10 s
             a_lab_the_size_of_a_postage_stamp_george_whitesides  26.3M  25.6G  1.65K    187 20130808 16 min 16 s
             a_test_for_parkinson_s_with_a_phone_call_max_little  12.5M  25.6G  1.65K    187 20130614 6 min 3 s
                 massive_scale_online_collaboration_luis_von_ahn  32.2M  25.6G  1.62K    184 20130808 16 min 39 s
                 a_short_intro_to_the_studio_school_geoff_mulgan  8.79M  25.6G  1.49K    169 20130816 6 min 16 s
           3_stories_of_local_eco_entrepreneurship_majora_carter  27.2M  25.7G  1.46K    165 20130725 16 min 34 s
                             put_a_value_on_nature_pavan_sukhdev  31.9M  25.7G  1.46K    165 20130726 16 min 30 s
                                   how_to_stop_torture_karen_tse  20.1M  25.7G  1.32K    150 20130809 8 min 44 s
    how_cyberattacks_threaten_real_world_peace_guy_philippe_gold  13.5M  25.7G  1.31K    149 20130808 9 min 23 s
               the_day_i_turned_down_tim_berners_lee_ian_ritchie  10.7M  25.7G  1.28K    145 20130809 5 min 41 s
    women_entrepreneurs_example_not_exception_gayle_tzemach_lemm  28.7M  25.8G  1.25K    142 20130815 13 min 16 s
                       the_arts_festival_revolution_david_binder  17.0M  25.8G  1.21K    137 20130517 9 min 4 s
                         filming_democracy_in_ghana_jarreth_merz  18.0M  25.8G  1.08K    123 20130809 8 min 36 s
                        there_are_no_scraps_of_men_alberto_cairo  36.4M  25.8G  1.07K    121 20130815 19 min 2 s
                   an_unexpected_place_of_healing_ramona_pierson  33.8M  25.9G  1.01K    114 20130815 11 min 12 s
                          digital_humanitarianism_paul_conneally  17.6M  25.9G  0.99K    113 20130626 10 min 57 s
                  the_economic_case_for_preschool_timothy_bartik  31.1M  25.9G    709   78.0 20130815 15 min 48 s
              let_s_crowdsource_the_world_s_goals_jamie_drummond  26.6M  25.9G    688   76.0 20130628 12 min 10 s



```python

```

* Enter the view count minimum, which will generate a list of wanted youtube id's.


```python
views_min = 120000
print(type(views_min))
wanted_ids = []
sql = 'SELECT yt_id from video_info where views_per_year > ?'
db.c.execute(sql,[views_min,])
rows = db.c.fetchall()
for row in rows:
    wanted_ids.append(row['yt_id'])
```

    <class 'int'>


* Now let's start building up the output directory



```python
import shutil
# copy the default top level directories (these were in the zim's "-" directory )
cpy_dirs = ['assets','cache','channels']
for d in cpy_dirs:
    if not os.isdir(os.path.join(OUTPUT_DIR,d))
        os.makedirs(os.path.join(OUTPUT_DIR,d))
    src = os.path.join(PROJECT_DIR,d)
    dest = os.path.join(OUTPUT_DIR,d)
    shutil.copytree(src,dest,dirs_exist_ok=True)
```


```python
# Copy the videos selected by the wanted_ids list to output file
for f in wanted_ids:
    if not os.path.isdir(os.path.join(OUTPUT_DIR,'-/videos',f)):
        os.makedirs(os.path.join(OUTPUT_DIR,'-/videos',f))
    src = os.path.join(PROJECT_DIR,'-/videos',f)
    dest = os.path.join(OUTPUT_DIR,'-/videos',f)
    shutil.copytree(src,dest,dirs_exist_ok=True)
```


```python
# Grab the meta data from the original zim "M" directory 
#   and create a script for zimwriterfs
def get_file_value(path):
    with open(path,'r') as fp:
        
        try:
            return fp.read()
        except:
            return ""
meta_file_names = []        


```


```python
# Copy the top level html files 
for f in iter(os.listdir(PROJECT_DIR)):
    if os.path.isfile(os.path.join(PROJECT_DIR,f)):
        src = os.path.join(PROJECT_DIR,f)
        !cp {src} {OUTPUT_DIR}
```


```python
# Write a new mapping from categories to vides (with some removed)
outstr = ''
for cat in zim_category_js:
    outstr += 'var json_%s = [\n'%cat
    for video in range(len(zim_category_js[cat])):
        if zim_category_js[cat][video].get('id','') in wanted_ids:
            outstr += json.dumps(zim_category_js[cat][video],indent=1)
            outstr += ','
    outstr = outstr[:-1]
    outstr += '];\n'
with open(OUTPUT_DIR + '/-/assets/data.js','w') as fp:
    fp.write(outstr)

```


```python
# Create a template for a script to run zimwriterfs
mk_zim_cmd = """
zimwriterfs --language=eng\
            --welcome=home.html\
            --favicon=./favicon.jpg\
            --title=teded_en_med\
            --description=\"TED-Ed Videos from YouTube Channel\"\
            --creator='Youtube Channel “TED-Ed”'\
            --publisher=IIAB\
            --name=TED-Ed\
             %s %s.zim"""%(OUTPUT_DIR,PROJECT_NAME)
with open(HOME + '/zimtest/' + PROJECT_NAME + '-zimwriter-cmd.sh','w') as fp:
    fp.write(mk_zim_cmd)
```


```python

```


```python

```