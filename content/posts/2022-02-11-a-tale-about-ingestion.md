+++
title = "🔊 A tale about ingestion"
date = "2022-02-11"
tags = ["talk", "elasticsearch"]
+++

## Abstract

Avant même d'effectuer une recherche dans Elasticsearch, vous devez l'alimenter avec vos données. Dans cet présentation, je passerai en revue tous les moyens que nous avons utilisés pour envoyer des millions de documents à nos index, des plus mauvais aux plus optimisés.

Mais même avec les optimisations, l'ingestion peut prendre du temps. Pendant ce temps, notre index en ligne pourrait avoir des opérations d'écriture que nous devons également répliquer sur notre nouvel index ! Je vous expliquerai donc aussi comment nous avons propagé ces changements aux deux index pour que tout aille bien une fois le nouvel index en ligne.

## Video

Video:
<video aria-label="ElasticCC: Une histoire d'ingestion" preload="metadata" crossorigin="anonymous" class="_1l-KU _3OgU3 _3-9KU" style="background-color: transparent;" playsinline="" data-testid="hls-video" tabindex="0" src="blob:https://play.vidyard.com/14143bc1-35cf-4059-a877-b0fbd5153eda"></video>

On Elastic website: https://community-conference.elastic.co/session/305686
