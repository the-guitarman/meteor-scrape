# ParseWebsite
This core module takes an HTML string and returns a proper data object. The
following NPM modules are used:

    # unfluff = Npm.require "unfluff" # not used currently
    franc = Npm.require "franc"
    articleTitle = Npm.require "article-title"
    teaser = Npm.require "teaser"
    summarize = Npm.require "summarizely"
    cheerio = Npm.require "cheerio"
    readability = Npm.require "readabilitySAX"

The API of this module includes a central `run()` method, and several small
helper methods for better testing and readability.

    @ParseWebsite = share.ParseWebsite =
      run: (html) ->
        txt = extractFromText html
        dom = extractFromDOM html
        mergeResults txt, dom

## Extract the raw data using some NPM modules

- [readability](https://www.npmjs.com/package/readabilitySAX) get the page text
- [franc](https://www.npmjs.com/package/franc) language detection
- [article-title](https://www.npmjs.com/package/article-title) title selector
- [teaser](https://www.npmjs.com/package/teaser) snippet
- [Summary](https://github.com/jbrooksuk/node-summary): TL;DR

In conjunction, these modules should be able to deliver a good starting point.

    extractFromText = (html) ->
      data = {}
      data.title = articleTitle html
      data.text = readability.process(html, {type: "text"}).text
      data.text = "" if /^under\s100\scharacters/i.test data.text
      data.text = "" if data.text.length < 50
      # NOTE: readability also finds the "next" page for paginated articles!
      # --> ToDo: follow the NEXT-pages and join all to one article!
      data.tags = Tags.findFrom "#{data.title} #{data.text}"
      data.teaser = new teaser(data).summarize() if data.text.length > 100
      data.teaser or= ""
      data.summary = summarize(data.text).join("\n")
      return data

## DOM Parsing

Every bit of valuable data is needed, so let's get dirty and fish for more
stuff by hand. To make scraping a bit more sane, the package
[cheerio](https://www.npmjs.com/package/cheerio) is used. It's kind of
jQuery, but on the server. This way, CSS3 selectors can be leveraged instead
of nasty XPaths or unreadable RegExps.

    extractFromDOM = (html) ->
      $ = cheerio.load html
      data = {}
      data.feeds = findFeeds $
      data.favicon = findFavicon $
      data.tags = findTags $
      data.references = findReferences $
      data.image = findImage $
      data.description = findDescription $
      data.text = findText $
      data.url = findCanonical $
      data.title = findTitle $
      return data

    findFeeds = ($) ->
      selector = """
        link[type='application/rss+xml'],
        link[type='application/atom+xml'],
        link[rel='alternate']
      """
      feeds = $(selector).map((i,e) -> $(this).attr("href")).get()
      return _.uniq _.compact feeds

    findFavicon = ($) ->
      selector = """
        link[rel='apple-touch-icon'],
        link[rel='shortcut icon'],
        link[rel='icon']
      """
      favicon = $(selector).attr("href")
      return favicon

    findTags = ($) ->
      str = $("meta[name='keywords']").attr("content")
      tags = []
      if str
        ary = if /;/.test str then str.split(";") else str.split(",")
        tags = _.uniq _.compact ary.map((s) -> s.trim().toLowerCase())
      return tags

    findReferences = ($) ->
      refs = $("a").map (i, e) -> $(e).attr("href")
      _.uniq _.compact _.filter refs, (r) -> Link.test r

    findImage = ($) ->
      selector = """
        meta[property='og:image'],
        meta[name='twitter:image']
      """
      list = $(selector).map((i,e) -> $(this).attr("content")).get()
      _.compact(list)[0]

    findDescription = ($) ->
      selector = """
        meta[property='og:description'],
        meta[name='twitter:description'],
        meta[name='description']
      """
      list = $(selector).map((i,e) -> $(this).attr("content")).get()
      _.compact(list)[0]

    findText = ($) ->
      $("h1,h2,h3,h4,h5,h6,p,article").map((i,e) -> $(e).text()).get().join("\n\n")

    findCanonical = ($) ->
      url = $("link[rel='canonical']").attr("href")
      if Link.test url then url else ""

    findTitle = ($) ->
      $("title,h1").first().text()

## Merge the results

Join the results from the TextParser and DomParser into one uniform
result object. Pick the best results if there is some overlap.

    mergeResults = (txt, dom) ->
      data = {}
      data.title = _(Text.clean(txt.title or dom.title)).prune 100
      data.text = Text.clean(txt.text or dom.text)
      data.lang = txt.lang
      data.description = _(Text.clean(dom.description or txt.teaser?.join(" ") or txt.summary)).prune 1000
      data.favicon = dom.favicon
      data.references = dom.references
      data.image = dom.image
      data.feeds = dom.feeds
      data.tags = Tags.clean _.union dom.tags, txt.tags
      data.lang = franc "#{data.title} #{data.text}"
      return data