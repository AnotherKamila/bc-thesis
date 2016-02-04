Project Deadlock (Bachelor's Thesis)
====================================

My Bachelor's Thesis: design and partial implementation of Project Deadlock.

Goal
----

The goal of the thesis is to design and implement the server for the access control system Deadlock. The thesis should contain:
- design of the server, which will manage the database of RFID card identifiers, mainly:
  - server architecture,
  - data structures for effective storing and processing of access rights,
  - communication protocol with devices controlling access,
  - solution for firmware updates for connected devices,
  - access logging,
- implementation in Python 3.

Notes:
------

TOC:

1. Introduction  
   blah blah
2. Specification  
   what do we want from the system?
3. High-level Design  
   specification => fun design choices => important decisions => overview of high-level structure  
   stuff for server & controller design (e.g. offline capabilities, fw updates, logging...)
4. Controller <--> Server Protocol  
   why => how
4. Server  
   1. Design  
      why
   2. Implementation  
      how
5. What's Next  
   future plans
6. Conclusion  
   it's deployed, it rocks, we're awesome (hopefully)

Templates and stuff
-------------------

Written in Markdown, compiled by Pandoc -- because I was sick of writing LaTeX but refused to have ugly typesetting. Template stolen from [github.com/tompollard/phd_thesis_markdown](https://github.com/tompollard/phd_thesis_markdown). Thesis LaTeX template for FMPH UK stolen from [compbio.fmph.uniba.sk/vyuka/bcinf](http://compbio.fmph.uniba.sk/vyuka/bcinf).
