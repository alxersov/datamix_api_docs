# Errors

The Datamix API will respond with the following error codes:

Error Code | Meaning
---------- | -------
403 | Forbidden -- You didn't include a valid, unexpired API key or session token.
404 | Not Found -- You probably hit the wrong URL. Check for typos.
500 | Internal Server Error -- We had a problem with our server. Try again later.
503 | Service Unavailable -- We're temporarily offline for maintenance. Please try again later.
