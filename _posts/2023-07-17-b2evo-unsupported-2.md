---
layout: post
tag: main
title: b2evo unsupported, endgueltig
subtitle: "b2evolution ist endgueltig tot. Wie gehts weiter? Wir machen jetzt Jekyll!"
date: 2023-07-17
author: eumel8
---

Es war mehr so eine Hassliebe all die Jahre, b2evolution und ich. 2007/2008 habe ich damit angefangen, neben Joomla auch b2evolution einzusetzen, vornehmlich zum Erstellen und Verwalten von Blogs. 4-5 Stück gabs am Ende zu unterschiedlichen Themen. Die Erstellung war einfach, das Hosting auf einem LAMP-Stack auch problemlos. Irgendwann wurden keine neueren MySQL-Versionen mehr unterstützt und dieser [Bug](https://github.com/b2evolution/b2evolution/issues/105) machte immer manuelle Eingriffe notwendig, die man natuerlich beim naechsten Update wieder vergessen hat. [Offiziell](https://b2evolution.net/news/2022/03/26/2022-update-eol) ist jetzt aber Schluss. Das letzte Release ist die 7.5.2-stable.

# Jekyll

[Jekyll](https://jekyllrb.com/) ist ein Static-Website-Generator geschrieben in Ruby. Klingt kompliziert, ist es aber nicht. Im Prinzip kann man Artikel zum Beispiel in Markdown Format verfassen, Jekyll baut daraus HTML-Dateien und liefert sie über

```bash
bundle exec jekyll serve -H 0.0.0.0
```

aus. Dazu bedarf es einer Datei `_config.yml` und `Gemfile` mit der zusaetzliche RubyGems geladen werden können. Mit dem Kommando `bundle install` werden diese installiert. 

# Github Pages

Mit [Github Pages](https://docs.github.com/de/pages/setting-up-a-github-pages-site-with-jekyll) kann man nun mit Jekyll erstellte Webseiten im Internet bereitstellen. Dabei wird automatisch das Generieren der HTML-Seiten übernommen, wenn man unter Settings im jeweiligen Github-Repo die Pages aktiviert. Die Seite ist dann automatisch unter der Adresse <username>.github.io/<reponame> erreichbar. Der Clou: Das ganze ist kostenlos. Und man kann sogar seine eigene Domain (etwa die vom alten Blog) hinterlegen.

# Migration

Im b2evolution CMS liegen die Artikel der Blogposts in der MySQL-Datenbank und die Bilder auf dem Dateisystem. Vielleicht fangen wir mit Letzterem an und sichern die Dateien, um diese im Gitrepo dann später auszupacken:

```bash
tar cvfz /data/media.tgz media/
```

Wenn man den Cache aktiviert hat, kann man die Cache-Dateien vorher noch löschen:

```bash
find media -name .evocache | xargs rm -rf
```

Wenn man sich mit der Datenbank verbindet, kann man erstmal rausfinden, wieviel Blogs wir im b2evolution haben:

```shell
select blog_name,blog_ID from evo_blogs;
```

Das ergibt dann eine Liste mit Namen und Id. Abgespeichert sind ie Blogbeitraege nach Kategorien. Ich benutze davon nicht allzuviele. Bislang gab es immer die Kategorie `main`. Aber wir können die Kategorie in Jekyll als `Tag` uebernehmen. Die Markdown Dateien haben alle einen Kopf, der mit einer Templatesprache verarbeitet wird. Dieses Script liest die Blogposts aus der Datenbank und erstellt Markdown Dateien daraus:

<details>
  <summary>migration.sh</summary>

  ```bash
  #!/bin/bash
  
  imagepath="/unsupported/media"
  blogdb="DBblog"
  blogid=6
  
  OIFS="$IFS"
  IFS=$'\n'
  oset="$-"
  set -f
  while IFS=$'\t' read -a cats; do
      unset IFS
      catname=${cats[0]}
      catidfull=${cats[1]}
      catid=$(echo $catidfull | sed 's/[^0-9]*//g')
      if [ ! -z $catid ]; then
          OIFS="$IFS"
          IFS=$'\n'
          oset="$-"
          set -f
          while IFS=$'\t' read -a post; do
              unset IFS
              datearray=(${post[2]})
              work="work.md"
              markdown="${datearray[0]}-${post[0]}.md"
              rm -f $markdown
              echo "---" >$work
              echo "layout: post" >>$work
              echo "tag: $catname" >>$work
              echo "title: ${post[1]}" >>$work
              echo "subtitle: \"${post[3]}\"" >>$work
              echo "date: ${datearray[0]}" >>$work
              echo "author: eumel8" >>$work
              echo "---" >>$work
              echo "" >>$work
              echo ${post[4]} >>$work
              sed -i 's/\rn/\n/g' $work
              while IFS= read -r post; do
                  if [[ $post = [image:* ]]; then
                      tmp=${post#*:}
                      c=${tmp%]*}
                      b=$(echo $c | sed 's/[^0-9]*//g')
                      image=$(mysql $blogdb -e "select ef.file_path from evo_files ef, evo_links el where ef.file_ID=el.link_file_ID and el.link_ID=$b")
                      imagearray=(${image})
                      echo "<img src=\"${imagepath}/${imagearray[1]}\" width=\"585\" height=\"386\"/>" >>$markdown
                  elif [[ $post = [teaserbreak* ]]; then
                      echo "<br/>" >>$markdown
                  else
                      echo "$post" >>$markdown
                  fi
              done <"$work"
          done < <(mysql ${blogdb} -e "select post_urltitle,post_title,post_datecreated,post_excerpt, post_content from evo_items__item where post_main_cat_ID=$catid;")
          rm -f $work
      fi
  done < <(mysql ${blogdb} -e "select cat_name,cat_ID from evo_categories where cat_blog_ID=$blogid;")
  ```
</details>


`imagepath` sollte man noch anpassen, ebenso die `blogdb` und `blogid`, aber im Grossen und Ganzen sollte man eine ansehnliche Liste von Markdown Dateien erstellt haben. Ob diese syntaktisch korrekt sind, kann man in Jekyll sofort ueberpruefen, wenn diese Dateien im `_posts` Ordner liegen und Jekyll die Dateien rendern kann. Etwaige Fehler wie Sonderzeichen werden ausgegeben und koennen manuell oder maschinell korrigiert werden, indem man das Script etwas anpasst.

# Extras

## Themes

Unser Blog brauch natuerlich auch ein Theme. Jekyll bietet da eine [reiche Auswahl](https://jekyllrb.com/docs/themes/), jedoch unterstuetzt Github Pages nur eine [Auswahl](https://pages.github.com/themes/). Behelfen kann man sich vielleicht mit einem schoenen Hintergrundbild von [Canva](https://www.canva.com/templates/?query=wallpaper). Mit eine Probe-Pro-Abo hat man Zugriff auf tausende Vorlagen.

## Suche

Im alte Blog gab es auch ein Suchmodul. Jetzt koennte man meinen, ein statischer Webseitengenerator kann sowas nicht. Weit gefehlt! Man muss sich nur etwas mit der Script-Sprache im Jekyll auseinandersetzen und mit etwas Javascript kann man eine erstaunlich gute Suchmaschine implementieren, beschrieben etwa [hier](https://blog.webjeda.com/instant-jekyll-search/) und als Beispiel mit Programmcode [hier](https://github.com/christian-fei/Simple-Jekyll-Search).

## Statistik/Logs

Logfiles gibt es bei Github Pages nicht zum Auswerten und auch Jekyll bietet sowas von Hause aus nicht an. Aber es gibt auch hier [Projekte wie Open-Web-Analytics](https://github.com/Open-Web-Analytics/Open-Web-Analytics). Dazu muss man in die `_includes/footer.html` einen Tracking-Code hinzufuegen, der jeden Seitenaufruf an den OWA Server puscht. Das Problem ist, dass dieser auch wieder PHP und MySQL benoetigt und wer sich im Zuge der Migration von LAMP trennen will, verwendet vielleicht besser [Google Analytics](https://analytics.google.com)

