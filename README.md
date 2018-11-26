"""routine to get video url from twitter popup windows"""

import asyncio
from pyppeteer import launch

url = 'https://twitter.com/Twitter'
screen_name = url.split('/')[-1] # may want to use this later

async def main():
    # on widnows pc, can change headless to False to see code in action
    browser = await launch(headless=True)
    page = await browser.newPage()

    # without try/except chromium processes will continue to run
    # will need to manually close them or pc will slow down considerably

    try:
        # interceptin requests to either see what is be requested
        # or to act on the request so that the page can load faster
        # this hellps to reduce timemout issues for long load times or slow pages
        await page.setRequestInterception(True)
        page.on('request', interceptedRequest)

        await page.goto(url, timeout=30000, waitUntil='networkidle0')

        # this is not needed in this project
        # await page.setViewport({ 'width': 1280, 'height': 926 })

        # for debugging and viewing html - print to file
        # ... note the encoding is needed, otherwise error
        page_html = await page.content()
        with open('page_html.txt', 'w', encoding='utf8') as f:
            f.write(page_html)

        # find the number of total tweets the user has
        # the class name has spaces in the html, to use the selector, use '.'
        # this acts as AND operator. Using ', ' separator is an OR and finds
        # too many elements
        # get to know CSS selectors
        selector_num_tweets = '.ProfileNav-item.ProfileNav-item--tweets.is-active > a > span:nth-child(3)'
        num_tweets = await page.querySelectorEval(selector_num_tweets,
                                           '(element) => element.getAttribute("data-count")')
        print('Total Number of Tweets: ', num_tweets)

        # find tweet urls for the first page load
        # this gets the user's tweets as well as retweets (other users' tweets)
        # this returns a list
        # looking in 'a' elements

        #-----------------------------------------------------------------------
        # without this section, only 20 tweet urls will be captured

        print()
        print('scrolling down....')
        print()

        # had to play with the scroll by amount and waitfor to get the next 20 tweets
        # NOTE: have not figured out how to scroll past the first 40

        await page.evaluate('window.scrollBy(0, 10000)')
            # 10000 gets another 20 tweets
        print('waitfor ....')
        await page.waitFor(5000)
        #-----------------------------------------------------------------------

        tweet_urls = await page.querySelectorAllEval('.tweet-timestamp.js-permalink.js-nav.js-tooltip',
                                                'e => e.map((a)=>a.href)')
        print('Number of tweets in list: ', len(tweet_urls))

        for tweet_url in tweet_urls:
            print(tweet_url)

        ###------------------------------------
        # this section uses more complex selector to exclude retweets
        # see the addition of [data-screen-name=' + screen_name + '] to the selector
        # note this is looking at different element than above
        # looking in 'div' elements
        class_sel_with_vid = '.tweet.js-stream-tweet.js-actionable-tweet.js-profile-popup-actionable.dismissible-content.original-tweet.js-original-tweet[data-screen-name=' + screen_name + ']'
        tweet_urls_a = await page.querySelectorAllEval(class_sel_with_vid,
                                                'e => e.map((div)=>div.getAttribute("data-permalink-path"))')
        print('Number of tweets in list for this user only: ', len(tweet_urls_a))
        for tweet_url_a in tweet_urls_a:
            print(tweet_url_a)
        ###------------------------------------

        # loop through tweet urls to get popup window's page_html and get video link
        # !!!!! NOTE: for some reason this crashes after a few items into the list
            # mostly get a 'connection is closed' error
        # creates a new incognito browser context for each tweet to scrape
        # use try/except to be sure chromium processes are killed if errors occur
        for tweet_url_a in tweet_urls_a:
            try:
                print('https://twitter.com' + tweet_url_a)
                popup_context = await browser.createIncognitoBrowserContext()
                popup_page = await popup_context.newPage()
                await popup_page.goto('https://twitter.com' + tweet_url_a, timeout=30000, waitUntil='networkidle0')

                popup_html = await popup_page.content()
                # for debugging and finding the elements you need
                with open('popup_html.txt', 'w', encoding='utf8') as f:
                    f.write(popup_html)

                # get video url from url if it exists
                # css selector used is based on attribute and it's value
                vid_url = await popup_page.querySelectorEval('[property="og:video:url"]',
                                                   '(element) => element.getAttribute("content")')
                print('Video URL: ', vid_url)

                # close the popup
                await popup_context.close()

            except Exception as e:
                # close the popup
                await popup_context.close()
                print(e)

        #________________________ other ______________________________________________
        # returns the text of content but not sure what it can do for me
        # content = await page.evaluate('document.body.textContent', force_expr=True)
        # print(content)

        # this works to get objects but not sure what to do with them
        # elements_objects = await page.querySelectorAll('.tweet-timestamp.js-permalink.js-nav.js-tooltip')
        # print(elements_objects)
        #________________________ other ______________________________________________

    except Exception as e:
        # close the browser
        await browser.close()
        print(e)

    # close the browser
    await browser.close()


async def interceptedRequest(req):
    """ intercept requests and act on them"""
    # print every request and it's type
    # print(f'Request: {req.resourceType} -- {req.url}')
    if req.resourceType == 'image':
    # abort image load requests
        # print(f'Request: {req.resourceType} aborted')
        # print(f'Request: {req.url}')
        await req.abort()
    else:
        # always continue if not aborting
        # print(f'Request: {req.resourceType} -- {req.url}')
        await req.continue_()

# start the scrape - call the main function
asyncio.get_event_loop().run_until_complete(main())
