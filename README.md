
The code and queries etc here are unlikely to be updated as my process
evolves. Later repos will likely have progressively different approaches
and more elaborate tooling, as my habit is to try to improve at least
one part of the process each time around.

---------

Step 1: Configure config.json
=============================

All the relevant metadata now lives in config.json: ideally nothing will
need tweaked after this. We need to be careful here to get the history
of Wikidata IDs for the constituency correct.

Step 1: Scrape the results
==========================

```sh
bundle exec ruby scraper.rb config.json | tee wikipedia.csv
```

Step 2: Check for missing party IDs
===================================

```sh
xsv search -v -s party 'Q' wikipedia.csv
```

Five to create:

* wb ce create-new-party.js "Highlander IV Wednesday Promotion"
* wb ce create-new-party.js "Ian For King"
* wb ce create-new-party.js "Alfred The Chicken"
* wb ce create-new-party.js "Rainbow Alliance"
* wb ce create-new-party.js "Chauvinist Raving Alliance"

Later update:

Turns out there are actually a few others that are mis-linked in
Wikipedia, so also need new items to change the links to:

* wb ce create-new-party.js "Sack Graham Taylor" (Q98600480)
* wb ce create-new-party.js "Buy the Daily Sport" (Q98600481)
* wb ce create-new-party.js "Save the National Health Service" (Q98600482)

Ste 3: Check for missing election IDs
=====================================

```sh
xsv search -v -s election 'Q' wikipedia.csv | xsv select electionLabel | uniq
```

Nothing missing.

Step 4: Generate possible missing person IDs
============================================

```sh
xsv search -v -s id 'Q' wikipedia.csv | xsv select name | tail +2 |
  sed -e 's/^/"/' -e 's/$/"@en/' | paste -s - |
  xargs -0 wd sparql find-candidates.js |
  jq -r '.[] | [.name, .item.value, .election.label, .constituency.label, .party.label] | @csv' |
  tee candidates.csv
```

Step 5: Combine Those and generate QuickStatements commands
===========================================================

```sh
xsv join -n --left 2 wikipedia.csv 1 candidates.csv | xsv select '10,1-8' | sed $'1i\\\nfoundid' | tee combo.csv
bundle exec ruby generate-qs.rb config.json | tee commands.qs
```

Then sent to QuickStatements as https://editgroups.toolforge.org/b/QSv2T/1598251726416/
