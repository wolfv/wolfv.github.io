---
layout: post
title: "Reverse Engineering the Siri Knowledge Endpoint"
author: "Wolf Vollprecht"
categories: posts
---

I've been curious how *Siri Knowledge* works and what data it receives from Apples
servers. As a long term goal, I am curious how feasible it is to implement something 
similar based on open data (and maybe even offline!) in order to weaken our dependency
on google and co (I have blogged [about this before]({% post_url 2019-10-16-open-data-and-search %}))

On a Mac OS X I have captured a couple of requests to the Apple servers, 
by using the neat [`mitmproxy`](https://mitmproxy.org/) tool to capture and analyze
the https traffic.

Here is the resulting Python script to query the knowledge API:

```py
import requests, sys

if len(sys.argv) > 1:
    q = sys.argv[1]
else:
    q = "bar"

params = {
	"24h": "1",
	"ab_seed": "25",
	"calendar": "gregorian",
	"cc": "FR",
	"esl": "en",
	"geosrc": "error.fail",
	"kb_ime": "en@com.apple.keylayout.US",
	"key": "XXXXXX",
	"locale": "en_FR",
	"q": q,
	"temp": "C",
	"time_zone": "Europe/Paris",
	"units": "SI"
}

headers = {
	"X-Apple-GeoSession": "XXXXXXXX667785981814708523897152837309518",  # can be left out
	"X-Apple-UI-Scale": "1.0",
	"X-Apple-Languages": '["en-FR"]',
	"Accept": "*/*",
	"X-Apple-UserGuid": "",  # doesn't seem to be used at all
	"User-Agent": "XXX",
	"Accept-Language": "en-us",
	"X-Apple-GeoMetadata": "XXXXXXXXX6IGByoFCMorEAA=",  # some base64 encoded geo data
	"Accept-Encoding": "gzip, deflate"
}

r = requests.get(
	'https://api-glb-par.smoot.apple.com/search',
	params=params,
	headers=headers
)

from pprint import pprint
pprint(r.json())
```

You will note that this doesn't work out of the box. That's because I have removed 
the `User-Agent` and `key` parameter. The reason is that this request fails if `key` 
and `User-Agent` are not set to corresponding pairs. However, I haven't uncovered the 
logic behind this so I'm not sure if this contains any confidential information 
e.g. about my Apple account or so. But you can easily get this data by sending some
queries through mitmproxy [(even from an iPhone)](https://medium.com/testvagrant/intercept-ios-android-network-calls-using-mitmproxy-4d3c94831f62).

If you insert the keys, you can run this from the command line like so:

`python3 ./knowledge.py "my query"`

## Query Results

The query results are actually quite interesting, especially the entities entry 
seems to map to Wikidata entities! For Barack Obama it's entities such as politician, 
lawyer, and so on. So it's pretty cool to see that Wikidata is used by Siri Knowledge.

The second example *tagesanz* is coming from a different data source, the 'web_index'.
Here Siri on iOS correctly suggests a website. This is one of the features that would be cool 
to recreate in a FOSS search engine in order to send fewer requests to Google when attempting 
to type a URL. A downloadable web index would be very cool! I imagine with 100 megabytes of 
webindex, one could store quite a few websites and meta data, and we could distill this
data set from Wikidata.

If you want to keep up to date with this, you can follow me on [Twitter](https://twitter.com/wuoulf/) and
feel free to send me comments over there.

---

What follows are the responses for a bunch of queries:

#### "Bara" (iOS)

```
[{'completion_score': 49199.0000124371,
  'duration': 53,
  'fbq': 'XXX',
  'partial_client_ip': 'X.X.X.X',
  'prefix': 'bara',
  'query': 'barack obama',
  'results': [{'block_id': 1,
               'canon_id': 'wiki:en.wikipedia.org/wiki/barack_obama',
               'completion': 'Barack Obama — Wikipedia',
               'ds_id': '<http://dbpedia.org/resource/Barack_Obama>',
               'engagement_score': 0,
               'entities': [{'probabilityScore': 1,
                             'topics': [{'identifier': 'Q82955',
                                         'score': 1.6043136},
                                        {'identifier': 'Q40348',
                                         'score': 0.8021568},
                                        {'identifier': 'Q188094',
                                         'score': 0.042135797},
                                        {'identifier': 'Q43229',
                                         'score': 0.033120677},
                                        {'identifier': 'Q35120',
                                         'score': 0.025347555}]},
                            {'name': 'Barack Obama', 'probabilityScore': 1}],
               'fbr': '',
               'filtered': False,
               'hide_rank': 15,
               'id': 'wiki:Barack+Obama',
               'is_quick_glance': True,
               'pb': 'Cg5TaXJpIEtub3dsZWRnZRrWBggXEoAGugH8BToMZGV0YWlsZWRfcm93mgPOAiIJCQAAAAAAAABAKgkJAAAAAAAA8D8yFgoJCQAAAAAAAERAEgkJAAAAAAAASUA6CmltYWdlL2pwZWdKEGlwaG9uZV93aWtpX2ljb25QAZgDNKID+QEK9gEK8wFodHRwczovL2Nkbi5zbW9vdC5hcHBsZS5jb20vaW1hZ2U/LnNpZz1JamxvUmNfN0NaOFBGS1YzaXRKdzd3JTNEJTNEJmRvbWFpbj13aWtpJmltYWdlX3VybD1odHRwcyUzQSUyRiUyRnVwbG9hZC53aWtpbWVkaWEub3JnJTJGd2lraXBlZGlhJTJGY29tbW9ucyUyRnRodW1iJTJGOCUyRjhkJTJGUHJlc2lkZW50X0JhcmFja19PYmFtYS5qcGclMkYyMDBweC1QcmVzaWRlbnRfQmFyYWNrX09iYW1hLmpwZyZzcGVjPTQwLTYwLU5DLTCqAxIKEAoMQmFyYWNrIE9iYW1hEAHKA/ECCgIQAyrqAgrnAgrkAkJhcmFjayBIdXNzZWluIE9iYW1hIElJIGlzIGFuIEFtZXJpY2FuIGF0dG9ybmV5IGFuZCBwb2xpdGljaWFuIHdobyBzZXJ2ZWQgYXMgdGhlIDQ0dGggcHJlc2lkZW50IG9mIHRoZSBVbml0ZWQgU3RhdGVzIGZyb20gMjAwOSB0byAyMDE3LiBBIG1lbWJlciBvZiB0aGUgRGVtb2NyYXRpYyBQYXJ0eSwgaGUgd2FzIHRoZSBmaXJzdCBBZnJpY2FuIEFtZXJpY2FuIHRvIGJlIGVsZWN0ZWQgdG8gdGhlIHByZXNpZGVuY3kuIEhlIHByZXZpb3VzbHkgc2VydmVkIGFzIGEgVS5TLiBzZW5hdG9yIGZyb20gSWxsaW5vaXMgZnJvbSAyMDA1IHRvIDIwMDggYW5kIGFuIElsbGlub2lzIHN0YXRlIHNlbmF0b3IgZnJvbSAxOTk3IHRvIDIwMDQu0gMPCg0KCVdpa2lwZWRpYRABMiYwX2Q4Yzg2NGQyLWM5NWItNDYxNi00ODU1LTBmZjNlMWRjYmQyODond2lraTplbi53aWtpcGVkaWEub3JnL3dpa2kvYmFyYWNrX29iYW1hYAE=',
               'placement': 'top',
               'qi_engagement_score': 0,
               'query': 'barack obama',
               'score': 0.8856840973468565,
               'section_bundle_id': 'com.apple.parsec.wiki',
               'section_header': 'Siri Knowledge',
               'section_key': 'wikipedia',
               'template': 'generic',
               'title': 'Barack Obama',
               'tophit': 0,
               'type': 'wiki',
               'url': 'https://en.wikipedia.org/wiki/Barack_Obama'}],
  'status': 'OK'}]
```

#### "tagesan"

```
[{'completion_score': 79099.0000009776,
  'duration': 22,
  'fbq': 'XXX',
  'partial_client_ip': 'X.X.X.X',
  'prefix': 'tagesan',
  'query': 'tagesanzeiger.ch',
  'results': [{'block_id': 1,
               'canon_id': 'wi:http://tagesanzeiger.ch',
               'completion': 'https://www.tagesanzeiger.ch',
               'engagement_score': 0,
               'fbr': '',
               'filtered': False,
               'hide_rank': 15,
               'id': 'wi:http://tagesanzeiger.ch',
               'pb': 'XXX',
               'placement': 'top',
               'qi_engagement_score': 0,
               'query': 'tagesanzeiger.ch',
               'score': 0.8175435529750685,
               'section_bundle_id': 'com.apple.parsec.web_index',
               'section_header': 'Siri Suggested Website',
               'section_key': 'web',
               'template': 'generic',
               'tophit': 0,
               'type': 'web_index',
               'url': 'https://www.tagesanzeiger.ch'}],
  'status': 'OK'}]
```

#### "Barack" (iOS)

```
[{'completion_score': 69999.0000002141,
  'duration': 26,
  'fbq': 'eyJwIjoiYmFyYWNrIiwicSI6ImJhcmFjayIsInRzIjoiMTU3NDAyMDUxNiIsImciOiJmci9pbGVkZWZyYW5jZS92aWxsZV9kZV9wYXJpcy9wYXJpcyIsImEiOiJzYWZhcmkiLCJkIjoiaXBob25lIiwibCI6ImVuX1VTIiwiaSI6IjAtMCwwIiwiYyI6IjQ4Ljg1NDMsMi4zNTI3IiwidCI6Wzg3LDc1LDY2LDEzMSwxMzUsMTgwLDQ0XSwiZHQiOls0NF0sInBtcXQiOls0NCwxODUsMzYsOTUsMTIwLDldLCJzaWQiOiJkMjhhOGQ1Ni0yODhjLTRiODEtN2E0Ny1hZTQ4NDFkZjMyMmEiLCJyIjoiMjAxOTExMTIuMTM0NDIzIiwidHAiOjAuMDEwMjE4OTc4LCJtcCI6MC43MTY2MjYxLCJxZCI6NCwiaXBzdCI6MTA2LCJpcHNmIjozNSwiYWJjMiI6MTQsImFiY2gxIjotMSwiYWJjaDIiOi0xfQ==',
  'partial_client_ip': 'X.X.X.X',
  'prefix': 'barack',
  'query': 'barack',
  'results': [{'block_id': 1,
               'canon_id': 'wiki:en.wikipedia.org/wiki/barack_(brandy)',
               'completion': 'Barack (brandy) — Wikipedia',
               'ds_id': '<http://dbpedia.org/resource/Barack_(brandy)>',
               'engagement_score': 0,
               'entities': [{'name': 'Barack', 'probabilityScore': 1}],
               'fbr': 'eyJkIjoid2lraSIsInIiOiJ3aWtpOkJhcmFjayslMjhicmFuZHklMjkiLCJjciI6Indpa2k6ZW4ud2lraXBlZGlhLm9yZy93aWtpL2JhcmFja18oYnJhbmR5KSIsImVpIjpbImh0dHBzOi8vZW4ud2lraXBlZGlhLm9yZy93aWtpL0JhcmFja18oYnJhbmR5KSJdLCJwbXF0IjpbNDQsMTg1LDM2LDk1LDEyMCw5XSwicCI6ImJhcmFjayIsInEiOiJiYXJhY2siLCJzaWQiOiJkMjhhOGQ1Ni0yODhjLTRiODEtN2E0Ny1hZTQ4NDFkZjMyMmEiLCJkbyI6MSwicm8iOjIsIm1vIjoxLCJzIjoiMCIsImNzIjpbMV19',
               'filtered': False,
               'hide_rank': 15,
               'id': 'wiki:Barack+%28brandy%29',
               'pb': 'Cg5TaXJpIEtub3dsZWRnZRrBBggXEugFugHkBToMZGV0YWlsZWRfcm93mgOwAiIJCQAAAAAAAABAKgkJAAAAAAAA8D8yFgoJCQAAAAAAAERAEgkJAAAAAACASkA6CmltYWdlL2pwZWdKEGlwaG9uZV93aWtpX2ljb25QAZgDNKID2wEK2AEK1QFodHRwczovL2Nkbi5zbW9vdC5hcHBsZS5jb20vaW1hZ2U/LnNpZz1GcWRVZ3B2VHlLUjN5SEYxSzkzM1F3JTNEJTNEJmRvbWFpbj13aWtpJmltYWdlX3VybD1odHRwcyUzQSUyRiUyRnVwbG9hZC53aWtpbWVkaWEub3JnJTJGd2lraXBlZGlhJTJGY29tbW9ucyUyRnRodW1iJTJGMiUyRjIwJTJGUGFsaW5rYS5qcGclMkYyMDBweC1QYWxpbmthLmpwZyZzcGVjPTQwLTYwLU5DLTCqAwwKCgoGQmFyYWNrEAHKA/0CCgIQAyr2AgrzAgrwAkJhcmFjayBpcyBhIHR5cGUgb2YgSHVuZ2FyaWFuIGJyYW5keSBtYWRlIG9mIGFwcmljb3RzOyBhbiBhcHJpY290IGJyYW5keS4gVGhlIHdvcmQgYmFyYWNrIGlzIGEgY29sbGVjdGl2ZSB0ZXJtIGZvciBib3RoIGFwcmljb3QgYW5kIHBlYWNoLiBOb3RlIHRoYXQgdGhlIEh1bmdhcmlhbiB3b3JkIGJhcmFjayBpcyBldHltb2xvZ2ljYWxseSByZWxhdGVkIHRvIHRoZSBFbmdsaXNoIHdvcmQgcGVhY2ggYXMgd2VsbCBhcyBtYW55IHdvcmRzIG1lYW5pbmcgdGhlIHNhbWUgaW4gbWFueSBFdXJvcGVhbiBsYW5ndWFnZXMsIGFuZCB1bHRpbWF0ZWx5IGdvZXMgYmFjayB0byBwZXJzaWtvcywgdGhlIGFuY2llbnQgR3JlZWsgd29yZCBmb3IgcGVyc2lhbicu0gMPCg0KCVdpa2lwZWRpYRABMiYwXzU5YmY1NTA1LTE3ZmItNDU5Yi01ZGYyLTE4MDViMjI0YmRjZToqd2lraTplbi53aWtpcGVkaWEub3JnL3dpa2kvYmFyYWNrXyhicmFuZHkpYAE=',
               'placement': 'top',
               'qi_engagement_score': 0,
               'query': 'barack',
               'score': 0.6303289636911701,
               'section_bundle_id': 'com.apple.parsec.wiki',
               'section_header': 'Siri Knowledge',
               'section_key': 'wikipedia',
               'template': 'generic',
               'title': 'Barack',
               'tophit': 0,
               'type': 'wiki',
               'url': 'https://en.wikipedia.org/wiki/Barack_(brandy)'}],
  'status': 'OK'}]

```

#### "Bar" (OS X Spotlight)

```
[{'completion_score': 39199.000083022,
  'duration': 67,
  'fbq': '',
  'partial_client_ip': 'X.X.X.X',
  'prefix': 'bar',
  'query': 'bar',
  'results': [{'app': {'bundle_id': 'com.apple.maps',
                       'install_url': '',
                       'name': 'Maps',
                       'punchout_uri': 'http://maps.apple.com/?q=bars&sll=48.854300%2C2.352700'},
               'block_id': 1,
               'canon_id': 'm:category:bars:0',
               'completion': '— La Chope des Compagnons',
               'completion_icon': {'h': 32, 'id': 'mac_maps_icon', 'w': 32},
               'engagement_score': 0,
               'fbr': 'XXX',
               'filtered': False,
               'hide_rank': 15,
               'icon': {'h': 32, 'id': 'mac_maps_icon', 'w': 32},
               'id': 'm:8878713025291995099:c',
               'maps_data': 'XXX',
               'maps_data_type': 'geo3.placedata.place',
               'maps_result_type': 'category',
               'qi_engagement_score': 0,
               'query': 'bars near me',
               'score': 0.8274983125118857,
               'section_bundle_id': 'com.apple.parsec.maps',
               'section_header': 'Maps',
               'section_header_more': 'Show all…',
               'section_header_more_url': 'http://maps.apple.com/?q=bars&sll=48.854300%2C2.352700',
               'section_key': 'maps',
               'should_enable_location': True,
               'template': 'maps',
               'title': 'La Chope des Compagnons',
               'tophit': 0,
               'type': 'maps'},
              {'app': {'bundle_id': 'com.apple.maps',
                       'install_url': '',
                       'name': 'Maps',
                       'punchout_uri': 'http://maps.apple.com/?q=bars&sll=48.854300%2C2.352700'},
               'block_id': 1,
               'canon_id': 'm:category:bars:1',
               'completion': '— Le Deep',
               'completion_icon': {'h': 32, 'id': 'mac_maps_icon', 'w': 32},
               'engagement_score': 0,
               'fbr': 'XXX',
               'filtered': False,
               'hide_rank': 15,
               'icon': {'h': 32, 'id': 'mac_maps_icon', 'w': 32},
               'id': 'm:979391342122842720:c',
               'maps_data': 'XXX',
               'maps_data_type': 'geo3.placedata.place',
               'maps_result_type': 'category',
               'qi_engagement_score': 0,
               'query': 'bars near me',
               'score': 0.8268169645523025,
               'section_bundle_id': 'com.apple.parsec.maps',
               'section_header': 'Maps',
               'section_header_more': 'Show all…',
               'section_header_more_url': 'http://maps.apple.com/?q=bars&sll=48.854300%2C2.352700',
               'section_key': 'maps',
               'should_enable_location': True,
               'template': 'maps',
               'title': 'Le Deep',
               'tophit': 0,
               'type': 'maps',
               'url': 'https://ledeep75004.spaces.live.com'},
              {'app': {'bundle_id': 'com.apple.maps',
                       'install_url': '',
                       'name': 'Maps',
                       'punchout_uri': 'http://maps.apple.com/?q=bars&sll=48.854300%2C2.352700'},
               'block_id': 1,
               'canon_id': 'm:category:bars:2',
               'completion': '— Barberousse',
               'completion_icon': {'h': 32, 'id': 'mac_maps_icon', 'w': 32},
               'engagement_score': 0,
               'fbr': 'XXX',
               'filtered': False,
               'hide_rank': 15,
               'icon': {'h': 32, 'id': 'mac_maps_icon', 'w': 32},
               'id': 'm:6374466807050099668:c',
               'maps_data': 'XXX',
               'maps_data_type': 'geo3.placedata.place',
               'maps_result_type': 'category',
               'qi_engagement_score': 0,
               'query': 'bars near me',
               'score': 0.8268169645523025,
               'section_bundle_id': 'com.apple.parsec.maps',
               'section_header': 'Maps',
               'section_header_more': 'Show all…',
               'section_header_more_url': 'http://maps.apple.com/?q=bars&sll=48.854300%2C2.352700',
               'section_key': 'maps',
               'should_enable_location': True,
               'template': 'maps',
               'title': 'Barberousse',
               'tophit': 0,
               'type': 'maps',
               'url': 'http://www.barberousse.com'},
              {'app': {'bundle_id': 'com.apple.maps',
                       'install_url': '',
                       'name': 'Maps',
                       'punchout_uri': 'http://maps.apple.com/?q=bars&sll=48.854300%2C2.352700'},
               'block_id': 1,
               'canon_id': 'm:category:bars:3',
               'completion': '— Le Saint Régis',
               'completion_icon': {'h': 32, 'id': 'mac_maps_icon', 'w': 32},
               'engagement_score': 0,
               'fbr': 'XXX',
               'filtered': False,
               'hide_rank': 15,
               'icon': {'h': 32, 'id': 'mac_maps_icon', 'w': 32},
               'id': 'm:5237282461431587876:c',
               'maps_data': 'XXX',
               'maps_data_type': 'geo3.placedata.place',
               'maps_result_type': 'category',
               'qi_engagement_score': 0,
               'query': 'bars near me',
               'score': 0.8268169645523025,
               'section_bundle_id': 'com.apple.parsec.maps',
               'section_header': 'Maps',
               'section_header_more': 'Show all…',
               'section_header_more_url': 'http://maps.apple.com/?q=bars&sll=48.854300%2C2.352700',
               'section_key': 'maps',
               'should_enable_location': True,
               'template': 'maps',
               'title': 'Le Saint Régis',
               'tophit': 0,
               'type': 'maps',
               'url': 'https://www.lesaintregis-paris.com'},
              {'block_id': 1,
               'canon_id': 'bing:https%3A%2F%2Ftheculturetrip.com%2Feurope%2Fnorway%2Farticles%2Fthe-10-best-bars-in-central-oslo%2F',
               'card_sections': [{'card_section_id': '0_1e2f41bc-f26f-40e1-5e29-9e10d859adef',
                                  'image': {'h': 100,
                                            'id': 'mac_NoImage-Safari-100-Spotlight-OSX',
                                            'is_template': True,
                                            'w': 100},
                                  'image_align': 'top',
                                  'result_id': 'bing:https%3A%2F%2Ftheculturetrip.com%2Feurope%2Fnorway%2Farticles%2Fthe-10-best-bars-in-central-oslo%2F',
                                  'subtitle': 'theculturetrip.com',
                                  'title': 'The 10 Best Bars In Central Oslo - '
                                           'Culture Trip',
                                  'type': 'rich_title'},
                                 {'card_section_id': '1_dd36720f-fa67-4f12-7e98-de2f126a1566',
                                  'description': '',
                                  'result_id': 'bing:https%3A%2F%2Ftheculturetrip.com%2Feurope%2Fnorway%2Farticles%2Fthe-10-best-bars-in-central-oslo%2F',
                                  'title': '9 Feb 2017 ... Read our guide to '
                                           'the 10 best places to enjoy a '
                                           'drink or two in Central Oslo, \n'
                                           'Norway.',
                                  'type': 'description'},
                                 {'card_section_id': '2_5dcbf502-4200-45eb-65ae-6e15583f4528',
                                  'key': 'Show Google Results',
                                  'key_nowrap': True,
                                  'result_id': 'bing:https%3A%2F%2Ftheculturetrip.com%2Feurope%2Fnorway%2Farticles%2Fthe-10-best-bars-in-central-oslo%2F',
                                  'type': 'row',
                                  'url': 'https://www.google.com/search?channel=aplaamd&client=safari&hl=en&q=bar&safe=active&source=a-app2',
                                  'value_image': {'h': 16,
                                                  'id': 'mac_safari_icon_small',
                                                  'w': 16}}],
               'completion': 'https://theculturetrip.com/europe/norway/articles/the-10-best-bars-in-central-oslo/',
               'descriptions': [{'formatted_text': [{'text': '9 Feb 2017 ... '
                                                             'Read our guide '
                                                             'to the 10 best '
                                                             'places to enjoy '
                                                             'a drink or two '
                                                             'in Central '
                                                             'Oslo, \n'
                                                             'Norway.',
                                                     'text_weight': 1}],
                                 'text_maxlines': 2}],
               'engagement_score': 0,
               'fbr': 'XXX',
               'filtered': False,
               'footnote': 'theculturetrip.com/…/the-10-best-bars-i…',
               'hide_rank': 15,
               'id': 'bing:https%3A%2F%2Ftheculturetrip.com%2Feurope%2Fnorway%2Farticles%2Fthe-10-best-bars-in-central-oslo%2F',
               'placement': 'bottom',
               'qi_engagement_score': 0,
               'query': 'bars near me',
               'score': 0.6679695338308604,
               'secondary_title': '— The 10 Best Bars In Central Oslo - '
                                  'Culture Trip',
               'section_bundle_id': 'com.apple.parsec.bing',
               'section_header': 'Websites',
               'section_header_more': 'Show all…',
               'section_header_more_url': 'https://www.google.com/search?channel=aplaamd&client=safari&hl=en&q=bar&safe=active&source=a-app2',
               'section_key': 'bing',
               'template': 'generic',
               'thumbnail': {'h': 16,
                             'id': 'mac_safari_icon_small',
                             'round_corner_pt': 2,
                             'round_corner_px': 2,
                             'w': 16},
               'title': 'theculturetrip.com',
               'title_maxlines': 2,
               'top_hit_lead_in': 'Websites',
               'tophit': 0,
               'type': 'bing',
               'url': 'https://theculturetrip.com/europe/norway/articles/the-10-best-bars-in-central-oslo/'},
              {'block_id': 1,
               'canon_id': 'bing:https%3A%2F%2Fwww.tripadvisor.com%2FRestaurants-g190479-zfg11776-Oslo_Eastern_Norway.html',
               'card_sections': [{'card_section_id': '0_9f6a1961-fd9d-4e6c-5e69-f2cc8ac19ec5',
                                  'image': {'content_type': 'image/jpeg',
                                            'h': 100,
                                            'round_corner_pt': 2,
                                            'round_corner_px': 2,
                                            'url': 'https://cdn.smoot.apple.com/image?.sig=a6gDIct90O4G-Npsy0Asxw%3D%3D&domain=bing&image_url=https%3A%2F%2Fis4-ssl.mzstatic.com%2Fimage%2Fthumb%2FPurple113%2Fv4%2Fe7%2F05%2Fbb%2Fe705bbd7-55e5-ae1d-7376-1b18636e17c8%2Fsource%2F100x100bb.jpg&spec=100-100-NC-0',
                                            'w': 100},
                                  'image_align': 'top',
                                  'result_id': 'bing:https%3A%2F%2Fwww.tripadvisor.com%2FRestaurants-g190479-zfg11776-Oslo_Eastern_Norway.html',
                                  'subtitle': 'tripadvisor.com',
                                  'title': 'THE BEST Bars & Pubs in Oslo - '
                                           'TripAdvisor',
                                  'type': 'rich_title'},
                                 {'card_section_id': '1_645a69ba-f0df-4b43-67b4-c87609aafa40',
                                  'description': '',
                                  'result_id': 'bing:https%3A%2F%2Fwww.tripadvisor.com%2FRestaurants-g190479-zfg11776-Oslo_Eastern_Norway.html',
                                  'title': 'Bars & Pubs in Oslo, Eastern '
                                           'Norway: Find TripAdvisor traveler '
                                           'reviews of Oslo \n'
                                           'Bars & Pubs and search by ... '
                                           '“London Pub & Club showed me a '
                                           'good time” · 16.',
                                  'type': 'description'},
                                 {'card_section_id': '2_3b6fb0f3-cc60-4b26-6229-e97f61bc5b58',
                                  'key': 'Show Google Results',
                                  'key_nowrap': True,
                                  'result_id': 'bing:https%3A%2F%2Fwww.tripadvisor.com%2FRestaurants-g190479-zfg11776-Oslo_Eastern_Norway.html',
                                  'type': 'row',
                                  'url': 'https://www.google.com/search?channel=aplaamd&client=safari&hl=en&q=bar&safe=active&source=a-app2',
                                  'value_image': {'h': 16,
                                                  'id': 'mac_safari_icon_small',
                                                  'w': 16}}],
               'completion': 'https://www.tripadvisor.com/Restaurants-g190479-zfg11776-Oslo_Eastern_Norway.html',
               'descriptions': [{'formatted_text': [{'text': 'Bars & Pubs in '
                                                             'Oslo, Eastern '
                                                             'Norway: Find '
                                                             'TripAdvisor '
                                                             'traveler reviews '
                                                             'of Oslo \n'
                                                             'Bars & Pubs and '
                                                             'search by ... '
                                                             '“London Pub & '
                                                             'Club showed me a '
                                                             'good time” · 16.',
                                                     'text_weight': 1}],
                                 'text_maxlines': 2}],
               'engagement_score': 0,
               'fbr': '',
               'filtered': False,
               'footnote': 'tripadvisor.com/Restaurants-g190479-zfg…',
               'hide_rank': 15,
               'id': 'bing:https%3A%2F%2Fwww.tripadvisor.com%2FRestaurants-g190479-zfg11776-Oslo_Eastern_Norway.html',
               'placement': 'bottom',
               'qi_engagement_score': 0,
               'query': 'bars near me',
               'score': 0.6244376132595046,
               'secondary_title': '— THE BEST Bars & Pubs in Oslo - '
                                  'TripAdvisor',
               'section_bundle_id': 'com.apple.parsec.bing',
               'section_header': 'Websites',
               'section_header_more': 'Show all…',
               'section_header_more_url': 'https://www.google.com/search?channel=aplaamd&client=safari&hl=en&q=bar&safe=active&source=a-app2',
               'section_key': 'bing',
               'template': 'generic',
               'thumbnail': {'content_type': 'image/jpeg',
                             'h': 100,
                             'id': 'mac_safari_icon_small',
                             'round_corner_pt': 2,
                             'round_corner_px': 2,
                             'url': 'https://cdn.smoot.apple.com/image?.sig=a6gDIct90O4G-Npsy0Asxw%3D%3D&domain=bing&image_url=https%3A%2F%2Fis4-ssl.mzstatic.com%2Fimage%2Fthumb%2FPurple113%2Fv4%2Fe7%2F05%2Fbb%2Fe705bbd7-55e5-ae1d-7376-1b18636e17c8%2Fsource%2F100x100bb.jpg&spec=100-100-NC-0',
                             'w': 100},
               'title': 'tripadvisor.com',
               'title_maxlines': 2,
               'top_hit_lead_in': 'Websites',
               'tophit': 0,
               'type': 'bing',
               'url': 'https://www.tripadvisor.com/Restaurants-g190479-zfg11776-Oslo_Eastern_Norway.html'},
              {'block_id': 1,
               'canon_id': 'bing:https%3A%2F%2Fwww.visitoslo.com%2Fen%2Frestaurants-nightlife%2Fbar-pub%2F',
               'card_sections': [{'card_section_id': '0_e68d0fb1-247f-48ef-4dcf-60a8b8332ca0',
                                  'image': {'h': 100,
                                            'id': 'mac_NoImage-Safari-100-Spotlight-OSX',
                                            'is_template': True,
                                            'w': 100},
                                  'image_align': 'top',
                                  'result_id': 'bing:https%3A%2F%2Fwww.visitoslo.com%2Fen%2Frestaurants-nightlife%2Fbar-pub%2F',
                                  'subtitle': 'visitoslo.com',
                                  'title': 'Bars and pubs in Oslo - VisitOslo',
                                  'type': 'rich_title'},
                                 {'card_section_id': '1_725b0833-a863-47a7-57ec-17833a3277b1',
                                  'description': '',
                                  'result_id': 'bing:https%3A%2F%2Fwww.visitoslo.com%2Fen%2Frestaurants-nightlife%2Fbar-pub%2F',
                                  'title': 'All over the city you find '
                                           'everything from Irish pubs to '
                                           'cocktail bars; downtown, \n'
                                           'Aker Brygge, ... In Oslo the '
                                           'nearest bar or pub is usually just '
                                           'around the corner.',
                                  'type': 'description'},
                                 {'card_section_id': '2_1ca0e6b2-afd9-42de-588b-48cae100dd95',
                                  'key': 'Show Google Results',
                                  'key_nowrap': True,
                                  'result_id': 'bing:https%3A%2F%2Fwww.visitoslo.com%2Fen%2Frestaurants-nightlife%2Fbar-pub%2F',
                                  'type': 'row',
                                  'url': 'https://www.google.com/search?channel=aplaamd&client=safari&hl=en&q=bar&safe=active&source=a-app2',
                                  'value_image': {'h': 16,
                                                  'id': 'mac_safari_icon_small',
                                                  'w': 16}}],
               'completion': 'https://www.visitoslo.com/en/restaurants-nightlife/bar-pub/',
               'descriptions': [{'formatted_text': [{'text': 'All over the '
                                                             'city you find '
                                                             'everything from '
                                                             'Irish pubs to '
                                                             'cocktail bars; '
                                                             'downtown, \n'
                                                             'Aker Brygge, ... '
                                                             'In Oslo the '
                                                             'nearest bar or '
                                                             'pub is usually '
                                                             'just around the '
                                                             'corner.',
                                                     'text_weight': 1}],
                                 'text_maxlines': 2}],
               'engagement_score': 0,
               'fbr': 'XXX',
               'filtered': False,
               'footnote': 'visitoslo.com/en/restaurants-nightlife/…',
               'hide_rank': 15,
               'id': 'bing:https%3A%2F%2Fwww.visitoslo.com%2Fen%2Frestaurants-nightlife%2Fbar-pub%2F',
               'placement': 'bottom',
               'qi_engagement_score': 0,
               'query': 'bars near me',
               'score': 0.612443973187887,
               'secondary_title': '— Bars and pubs in Oslo - VisitOslo',
               'section_bundle_id': 'com.apple.parsec.bing',
               'section_header': 'Websites',
               'section_header_more': 'Show all…',
               'section_header_more_url': 'https://www.google.com/search?channel=aplaamd&client=safari&hl=en&q=bar&safe=active&source=a-app2',
               'section_key': 'bing',
               'template': 'generic',
               'thumbnail': {'h': 16,
                             'id': 'mac_safari_icon_small',
                             'round_corner_pt': 2,
                             'round_corner_px': 2,
                             'w': 16},
               'title': 'visitoslo.com',
               'title_maxlines': 2,
               'top_hit_lead_in': 'Websites',
               'tophit': 0,
               'type': 'bing',
               'url': 'https://www.visitoslo.com/en/restaurants-nightlife/bar-pub/'}],
  'sqf': 'XXX',
  'status': 'OK',
  'suggestions': [{'fbr': 'XXX',
                   'score': 8.302200149046257e-05,
                   'suggestion': 'bars near me'},
                  {'fbr': 'XXX',
                   'score': 7.457390165654942e-05,
                   'suggestion': 'barcelona'}]}]

```