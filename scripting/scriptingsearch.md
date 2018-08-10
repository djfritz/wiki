# Orchestration and Scripting Searches

Gravwell provides a robust scripting engine in which you can run searches, update resources, send alerts, or take action.  The orchestration engine allows for automating the tedious steps in an investigation and taking action based on search results without the need to involve a human.  

Orchestration scripts can be run [on a schedule](scheduledsearch.md) or by hand from the [command line client](#!cli/cli.md). Because the CLI allows the script to be re-executed interactively, we recommend developing and testing scripts in the CLI before creating a scheduled search.

## Built-in functions

Scripted searches can use built-in functions that mostly match those available for the [anko](#!scripting/anko.md) module, with some additions for launching and managing searches. The functions are listed below in the format `functionName(<functionArgs>) <returnValues>`.

### Resources and persistent data

* `getResource(name) []byte, error` returns the slice of bytes is the content of the specified resource, while the error is any error encountered while fetching the resource.
* `setResource(name, value) error` creates (if necessary) and updates a resource named `name` with the contents of `value`, returning an error if one arises.
* `setPersistentMap(mapname, key, value)` stores a key-value pair in a map which will persist between executions of a scheduled script.
* `getPersistentMap(mapname, key) value` returns the value associated with the given key from the named persistent map.
* `delPersistentMap(mapname, key)` deletes the specified key/value pair from the given map.

### Search entry manipulation

* `setEntryEnum(ent, key, value)` sets an enumerated value on the specified entry.
* `getEntryEnum(ent, key) value, error` reads an enumerated value from the specified entry.
* `delEntryEnum(ent, key)` deletes the specified enumerated value from the given entry.

### General utilities

* `len(val) int` returns the length of val, which can be a string, slice, etc.
* `toIP(string) IP` converts string to an IP, suitable for comparing against IPs generated by e.g. the packet module.
* `toMAC(string) MAC` converts string to a MAC address.
* `toString(val) string` converts val to a string.
* `toInt(val) int64` converts val to an integer if possible. Returns 0 if no conversion is possible.
* `toFloat(val) float64` converts val to a floating point number if possible. Returns 0.0 if no conversion is possible.
* `toBool(val) bool` attempts to convert val to a boolean. Returns false if no conversion is possible. Non-zero numbers and the strings “y”, “yes”, and “true” will return true.
* `typeOf(val) type` returns the type of val as a string, e.g. “string”, “bool”.

### Search management

Due to the way Gravwell's search system works, some of the functions in this section return Search structs (written as `search` in the parameters) while others return search IDs (written as `searchID` in the parameters). Each Search struct contains a search ID which can be accessed as `search.ID`.

Search structs are used to actively read entries from a search, while search IDs tend to refer to inactive searches to which we may attach or otherwise manage.

* `startBackgroundSearch(query, start, end) (search, err)` creates a backgrounded search with the given query string, executed over the time range specified by 'start' and 'end'. The return value is a Search struct. These time values should be specified using the time library; see the examples for a demonstration.
* `startSearch(query, start, end) (search, err)` acts exactly like `startBackgroundSearch`, but does not background the search.
* `detachSearch(search)` detaches the given search (a Search struct). This will allow non-backgrounded searches to be automatically cleaned up and should be called whenever you're done with a search.
* `attachSearch(searchID) (search, error)` attaches to the search with the given ID and returns a Search struct which can be used to read entries etc.
* `getSearchStatus(searchID) (string, error)` returns the status of the specified search, e.g. "SAVED".
* `getAvailableEntryCount(search) (uint64, bool, error)` returns the number of entries that can be read from the given search, a boolean specifying if the search is complete, and an error if anything went wrong.
* `getEntries(search, start, end) ([]SearchEntry, error)` pulls the specified entries from the given search. The bounds for `start` and `end` can be found with the `getAvailableEntryCount` function.
* `isSearchFinished(search) (bool, error)` returns true if the given search is complete
* `executeSearch(query, start, end) ([]SearchEntry, error)` starts a search, waits for it to complete, retrieves up to ten thousand entries, detatches from search and returns the entries.
* `deleteSearch(searchID) error` deletes the search with the specified ID
* `backgroundSearch(searchID) error` sends the specified search to the background; this is useful for "keeping" a search for later manual inspection.
* `downloadSearch(searchID, format, start, end) ([]byte, error)` downloads the given search as if a user had clicked the 'Download' button in the web UI. `format` should be a string containing either "json", "csv", "text", "pcap", or "lookupdata" as appropriate. `start` and `end` are time values.

### Sending results

* `httpGet(url) (string, error)` performs an HTTP GET request on the given URL, returning the response body as a string.
* `httpPost(url, contentType, data)` performs an HTTP POST request to the given URL with the specified content type (e.g. "application/json") and the given data as the POST body.
* `sendMail(hostname, port, username, password, from, to, subject, message) error` sends an email via SMTP. `hostname` and `port` specify the SMTP server to use; `username` and `password` are for authentication to the server. The `from` field is simply a string, while the `to` field should be a slice of strings containing email addresses. The `subject` and `message` fields are also strings which should contain the subject line and body of the email.
* `sendMailTLS(hostname, port, username, password, from, to, subject, message, disableValidation) error` sends an email via SMTP using TLS. `hostname` and `port` specify the SMTP server to use; `username` and `password` are for authentication to the server. The `from` field is simply a string, while the `to` field should be a slice of strings containing email addresses. The `subject` and `message` fields are also strings which should contain the subject line and body of the email.  The disableValidation argument is a boolean which disables TLS certificate validation.  Setting disableValidation to true is insecure and may expose the email client to man-in-the-middle attacks.


## An example script

This script creates a backgrounded search that finds which IPs have communicated with Cloudflare's 1.1.1.1 DNS service over the last day. If no results are found, the search is deleted, but if there are results the search will remain for later perusal by the user in the 'Manage Searches' screen of the GUI.

```
# Import the time library
var time = import("time")
# Define start and end times for the search
start = time.Now().Add(-24 * time.Hour)
end = time.Now()
# Launch the search
s, err = startSearch("tag=netflow netflow Dst==1.1.1.1 Src | unique Src | table Src", start, end)
if err != nil {
	return err
}
printf("s.ID = %v\n", s.ID)
# Wait until the search is finished
for {
	f, err = isSearchFinished(s)
	if err != nil {
		return err
	}
	if f {
		break
	}
	time.Sleep(1 * time.Second)
}
# Find out how many entries were returned
c, _, err = getAvailableEntryCount(s)
if err != nil {
	return err
}
printf("%v entries\n", c)
# If no entries returned, delete the search
# Otherwise, background it
if c == 0 {
	deleteSearch(s.ID)
} else {
	err = backgroundSearch(s.ID)
	if err != nil {
		return err
	}
}
# Always detach from the search at the end of execution
detachSearch(s)
```

To test the script, we can paste the contents into a file, say `/tmp/script`. We then use the Gravwell CLI tool to run the script. It will prompt to re-run as many times as desired and re-read the file for each run; this makes it easy to test changes to the script.

```
$ gravwell -s gravwell.example.org watch script
Username: <user>
Password: <password>
script file path> /tmp/script
0 entries
deleting 910782920
Hit [enter] to re-run, or [q][enter] to cancel

s.ID = 285338682
1 entries
Hit [enter] to re-run, or [q][enter] to cancel
```

The example above shows that the script was run twice; in the first run, no results were found and the search was deleted. In the second, 1 entry was returned, so the search was left intact.

Take care to delete old searches when testing scripts at the CLI; the scheduled search system will automatically delete old searches for you, but not the CLI.

## Using HTTP in scripts

Many modern computer systems use HTTP requests to trigger actions. Gravwell scripts offer the following basic HTTP operations:

* httpGet(url)
* httpPost(url, contentType, data)

We can use these functions and the JSON library included with Anko to modify the previous example script. Rather than backgrounding the search for later perusal, the script will build a list of IPs which have communicated with 1.1.1.1, encode them as JSON, and perform an HTTP POST to submit them to a server:

```
var time = import("time")
var json = import("encoding/json")
var fmt = import("fmt")

# Launch search
start = time.Now().Add(-24 * time.Hour)
end = time.Now()
s, err = startSearch("tag=netflow netflow Dst==1.1.1.1 Src | unique Src", start, end)
if err != nil {
	return err
}
# wait for search to finish
for {
	f, err = isSearchFinished(s)
	if err != nil {
		return err
	}
	if f {
		break
	}
	time.Sleep(1*time.Second)
}
# Get number of entries returned
c, _, err = getAvailableEntryCount(s)
if err != nil {
	return err
}
# Return if no entries
if c == 0 {
	return nil
}
# fetch the entries
ents, err = getEntries(s, 0, c)
if err != nil {
	return err
}
# Build up a list of IPs
ips = []
for i = 0; i < len(ents); i++ {
	src, err = getEntryEnum(ents[i], "Src")
	if err != nil {
		continue
	}
	ips = ips + fmt.Sprintf("%v", src)
}
# Encode IP list as JSON
encoded, err = json.Marshal(ips)
if err != nil {
	return err
}
# Post to an HTTP server
httpPost("http://example.org:3002/", "application/json", encoded)
detachSearch(s)
```