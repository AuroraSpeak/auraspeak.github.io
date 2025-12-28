+++ 
draft = false
date = 2025-12-27T20:25:42+01:00
title = "The Blog"
description = "How am I, what am I building and Some Rules"
slug = "the-blog"
authors = ["22peacemaker"]
externalLink = ""
series = []
+++

# A new beginning

Hi,

ich bin Jannis Karpinski ein Pädagoge in der Jugendhilfe aus Deutschland. Seit meiner Jugend programmiere ich und habe viele Projekte nicht beendet (siehe mein Github). Seit einiger zeit habe ich mich etwas in golang verliebt und nutze seit dem an eigentlich nur noch diese Sprache. Irgendwann hat sichin meinem Kopf der Gedanke festgesetzt, dass ich nur das teile was ich will. Nicht mehr und nicht weniger. Im besonderen open source hat es mir also angetan und seit dem versuche ich hier etwas zu reißen.

## What is this Blog about

In diesem Blog werde ich meine Reise beschreiben ein Dezentrales Voice Chat system aufzubauen. Es soll sicher sein aber auch von der Community getragen werden, ich selber habe nicht alleine die Mittel all das zu hosten, weshalb, sollte es mal zu einem Release kommen, nur kleine Kernsysteme von mir gehostet werden und der rest von der Community gehostet werden soll.

## Goals and Structure of the Blog

Der Blog soll Konzepte und Prinzipien verständlich erklären aber auch implementierungs Ideen geben und auch entscheidungen Begründen.

Er soll auch für mich eine referenz sein, damit ich mich an alles erinnere.
Daraus ergibt sich folgende reinfolge:
1. Erklärung der Konzepte bis zu dem Punkt wo ich sie verstanden habe.
2. Erklären warum ich diese Methode nutze
3. Es Implementieren step für step

Das hier wird kein go oder Crypto Grundkurs, prinzipien, die ich selber nicht verstanden habe werden eventuell erklärt und ich werde prinzipien, die ich selber nicht verstanden habe nochmal näher beleuchten. Damit möchte ich stolpersteine aus dem Weg räumen.

Leiter kann ich nicht jede einzelne zeile hier teilen, deshalb werde ich immer wieder auf das networking repo verweisen. Dort könnt ihr alles nachschauen.

## What could happen

Ich zeige euch nicht den optimalsten weg etwas zu schreiben. Nicht den richtigen weg etwas zu implementieren. Ich bin der überzeugung, dass es diesen nicht gibt, frei nach dem Motto: "Viele wege führen nach Rom".

Der Blog könnte einfach abbrechen. Wie gesagt, programmieren ist "nur" mein Hobby und auch ich kann irgendwann keine zeit oder lust mehr haben. Das ist aber nicht mein ziel nur eine realistische einschätzung (Pädagoge halt).

Der Code kann natürlich auch fehler beinhalten oder Kritische sicherheitslücken öffnen, dann dürft ihr mich einfach ansschreiben oder auch ganz selber am Code mitarbeiten. Fühlt euch frei!

## Why this way?

Dafür gibt es drei ganz einfache gründe:
1. Ich lerne das alles gerade erst selber es ist also irgendwo ein gemeinsames Lernen
2. Ich will es einfach machen, die Magie herrausnehemen. Jeder soll es verstehen können und selber nachbauen können.
3. Jeder soll die Möglichkeit haben es selber zu bauen. 

## Why this Project

Auch das ist relativ einfach. Man stößt immer wieder auf zwei namen wenn man einen Voice Server haben will. Teamspeak oder Discord. Hinter beidem stecken Firmen, die Geld verdienen will. Beide machen es auf anderem Wege. Teamspeak verkauft Server Discord Daten. Wenn man nun sagt, dann nehme ich doch einfach Teamspeak, wird einem relativ schnell darauf stoßen, dass Teamspeak einfach ein paar Features fehlen. Da mich das alles irgendwie nervt will ich versuchen es selber zu bauen. Auch wenn das irgendwie unmöglich errscheint, ist mein leitspruch: "Dann hat man einfach noch nicht genug rum gebastelt".

---

Aber jetzt lasst uns das coden Starten!