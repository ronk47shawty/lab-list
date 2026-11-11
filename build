"""Lab List build: pulls the latest NBA box scores + schedule and writes index.html.
Usage: python build.py [--embed]   (--embed inlines photos/logos as data URIs)"""
import pandas as pd, json, sys, io, base64, datetime, urllib.request, concurrent.futures, os
EMBED='--embed' in sys.argv
TZ='America/Toronto'
REL='https://github.com/sportsdataverse/sportsdataverse-data/releases/download'
today=pd.Timestamp.now(tz=TZ)
cur=today.year+1 if today.month>=8 else today.year   # season label = end year
def pq(url):
    with urllib.request.urlopen(url,timeout=120) as r: return pd.read_parquet(io.BytesIO(r.read()))
box=pd.concat([pq(f'{REL}/espn_nba_player_boxscores/player_box_{y}.parquet') for y in (cur-1,cur)])
sch=pq(f'{REL}/espn_nba_schedules/nba_schedule_{cur}.parquet')
valid=(set(sch.home_abbreviation)|set(sch.away_abbreviation))&set(box[box.season==cur].team_abbreviation)
box=box[box.team_abbreviation.isin(valid)&box.opponent_team_abbreviation.isin(valid)].copy()
box['game_date']=pd.to_datetime(box.game_date).dt.strftime('%Y-%m-%d')
box=box.sort_values(['game_date','game_id'])
played=box[(box.did_not_play==False)&(box.minutes.fillna(0)>0)]
tl=sorted(valid); ti={t:i for i,t in enumerate(tl)}
tinfo={}
for _,r in box.drop_duplicates('team_abbreviation').iterrows():
    tinfo[r.team_abbreviation]=[r.team_abbreviation,r.team_display_name,r.team_short_display_name,(r.team_color if isinstance(r.team_color,str) and r.team_color else '555555'),r.team_logo]
pl=played.groupby('athlete_id').last()
pids=list(pl.index); pi={p:i for i,p in enumerate(pids)}
players=[[r.athlete_display_name,ti[r.team_abbreviation],r.athlete_position_abbreviation or '',str(r.athlete_jersey or '')] for p,r in pl.iterrows()]
g=box.drop_duplicates('game_id'); gids=list(g.game_id); gi={x:i for i,x in enumerate(gids)}
games=[[r.game_date,int(r.season),int(r.season_type)] for _,r in g.iterrows()]
n=lambda x:int(x) if pd.notna(x) else 0
rows=[[gi[r.game_id],pi[r.athlete_id],ti[r.team_abbreviation],ti[r.opponent_team_abbreviation],1 if r.home_away=='home' else 0,1 if r.team_winner else 0,n(r.team_score),n(r.opponent_team_score),n(r.minutes),n(r.points),n(r.rebounds),n(r.assists),n(r.steals),n(r.blocks),n(r.turnovers),n(r.three_point_field_goals_made),n(r.three_point_field_goals_attempted),n(r.field_goals_made),n(r.field_goals_attempted),n(r.free_throws_made),n(r.free_throws_attempted),n(r.plus_minus),1 if r.starter else 0] for r in played.itertuples()]
# DNP rows (players who logged games) + reasons
reasons=[];ri={}
def rid(s):
    s=(s if isinstance(s,str) else '').strip().upper() or 'DID NOT PLAY'
    if s not in ri: ri[s]=len(reasons); reasons.append(s)
    return ri[s]
dn=box[(box.athlete_id.isin(pi))&((box.did_not_play==True)|(box.minutes.fillna(0)==0))]
dnp=[[gi[r.game_id],pi[r.athlete_id],ti[r.opponent_team_abbreviation],1 if r.home_away=='home' else 0,rid(r.reason)] for r in dn.itertuples()]
# status: consecutive current-season team games missed (DNP or absent from box)
present={}
for r in box.itertuples(): present.setdefault(r.game_id,{})[r.athlete_id]=(r.did_not_play==False and (r.minutes or 0)>0, r.reason)
tgames={}
for r in box[box.season==cur].drop_duplicates(['game_id','team_abbreviation']).itertuples(): tgames.setdefault(r.team_abbreviation,[]).append(r.game_id)
status={}
for p,r in pl.iterrows():
    if int(r.season)!=cur: continue
    gl=tgames.get(r.team_abbreviation,[]); streak=0; why=None
    for gid in reversed(gl):
        ok,rs=present.get(gid,{}).get(p,(False,None))
        if ok: break
        streak+=1; why=why if why is not None else rs
    if streak: status[pi[p]]=[streak,rid(why if isinstance(why,str) else 'INACTIVE')]
s=sch.copy(); s['gd']=pd.to_datetime(s.game_date_time).dt.tz_convert(TZ)
lo=today.strftime('%Y-%m-%d'); hi=(today+pd.Timedelta(days=21)).strftime('%Y-%m-%d')
s=s[(s.gd.dt.strftime('%Y-%m-%d')>=lo)&(s.gd.dt.strftime('%Y-%m-%d')<=hi)]
sched=[[r.gd.strftime('%Y-%m-%d'),r.gd.strftime('%-I:%M %p'),ti[r.away_abbreviation],ti[r.home_abbreviation]] for _,r in s.sort_values('gd').iterrows() if r.home_abbreviation in ti and r.away_abbreviation in ti]
data={'asof':played.game_date.max(),'built':today.strftime('%b %-d, %-I:%M %p ET'),'teams':[tinfo[t][:4] for t in tl],'players':players,'games':games,'rows':rows,'dnp':dnp,'reasons':reasons,'status':status,'sched':sched}
photo_url=lambda p:f'https://a.espncdn.com/combiner/i?img=/i/headshots/nba/players/full/{int(p)}.png&w=240&h=176'
logo_url=lambda t:'https://a.espncdn.com/combiner/i?img='+tinfo[t][4].split('espncdn.com')[1]+'&w=96&h=96'
photos={};logos={}
if EMBED:
    from PIL import Image
    def fetch(u):
        try:
            with urllib.request.urlopen(u,timeout=30) as r: return Image.open(io.BytesIO(r.read())).convert('RGBA')
        except Exception: return None
    def enc(im,bg):
        b=io.BytesIO(); base=Image.new('RGBA',im.size,bg); base.alpha_composite(im)
        (base if bg[3]==0 else base.convert('RGB')).save(b,'WEBP',quality=74); return 'data:image/webp;base64,'+base64.b64encode(b.getvalue()).decode()
    with concurrent.futures.ThreadPoolExecutor(16) as ex: imgs=list(ex.map(fetch,[photo_url(p) for p in pids]))
    for i,im in enumerate(imgs):
        if im is None or im.getbbox() is None: continue
        w,h=im.size; im=im.crop(((w-h)//2,0,(w-h)//2+h,h)).resize((96,96),Image.LANCZOS); photos[i]=enc(im,(64,66,74,255))
    with concurrent.futures.ThreadPoolExecutor(16) as ex: limgs=list(ex.map(fetch,[logo_url(t) for t in tl]))
    for i,im in enumerate(limgs):
        if im is not None: logos[i]=enc(im.resize((64,64),Image.LANCZOS),(0,0,0,0))
else:
    photos={i:photo_url(p) for i,p in enumerate(pids)}
    logos={i:logo_url(t) for i,t in enumerate(tl)}
j=lambda o:json.dumps(o,separators=(',',':')).replace('</','<\\/')
html=open(os.path.join(os.path.dirname(os.path.abspath(__file__)),'template.html')).read()
html=html.replace('__DATA__',j(data)).replace('__PHOTOS__',j(photos)).replace('__LOGOS__',j(logos)).replace('__REMOTE__','false' if EMBED else 'true')
out=sys.argv[sys.argv.index('--out')+1] if '--out' in sys.argv else 'index.html'
open(out,'w').write(html)
print(out,len(html),'rows',len(rows),'dnp',len(dnp),'out',len(status),'photos',len(photos),'logos',len(logos),'asof',data['asof'])
