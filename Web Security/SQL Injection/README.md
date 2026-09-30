# SQL INJECTION


In this level the goal was SQL injection again, but with a twist: this one was blind. Normally SQLi lets you pull data straight back — through a UNION query, an error message, something visible. Here, none of that existed. The server would never print the flag, or the password, or anything resembling database output. All I'd ever get back was a page and a status code, so any information I extracted would have to come from *inferring* something indirect, not reading it directly. 

For the first while I had basically no plan. I threw the usual SQLi patterns at it, the kind that would normally dump a table or bypass a login outright. And predictably got nowhere, because none of them were built for a target that gives you no output channel at all. 

The turning point was noticing something in how the server responded to a payload shaped like this.

```
1' OR (username='admin' AND SUBSTR(password, 1, N)='guess') --
```

`SUBSTR(password, 1, N)` pulls the first `N` characters of the real password. Comparing that slice against a guessed string of the same length is a way of asking the database a strict yes-or-no question: "do the first `N` characters of the password exactly equal this guess?" If they do, the whole `OR` condition becomes true for the admin row, and the login logic treats it as a valid authentication, a `302` redirect, the kind you'd normally see right after a successful login. If the guess is wrong, the condition is false, and the server comes back with a `403` instead. That status code *is* the leak. It's not the password itself, but it's a single bit of information every time I ask, and a single bit is enough if you're willing to ask enough questions. 

Which is exactly the catch: there's no way to do this by hand. Every character of the flag needs its own round of guesses against a charset, and a charset with even a modest 60-something characters times a flag that could easily run 30+ characters long is thousands of requests. I had zero prior experience writing an actual exploit script, so putting one together took a while, but the logic itself isn't complicated once the oracle is understood, it's just automating the same question over and over, one character position at a time.

```python
import requests 
url = "http://challenge.localhost/" 
known = "pwn" 
charset = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/={}_-." 

while not known.endswith("}"): 
for c in charset: 
guess = known + c 
inj = f"1' OR (username='admin' AND SUBSTR(password, 1, {len(guess)})='{guess}') -- " 
payload = { 
"username": "admin", 
"password": inj, 
} 
response = requests.post(url, data=payload, allow_redirects=False) 
if response.status_code == 302: 
known += c 
print(known) 
break 
```

`known` starts as `"pwn"` since that much of the flag format is a given. The outer `while` loop keeps going until the extracted string ends in `}`, the flag's closing character, that's the natural stopping condition, since there's no other signal telling the script when it's done. Inside, the `for` loop walks the whole charset, appending each candidate character to what's already confirmed and firing off the `SUBSTR` oracle query for that exact length. `allow_redirects=False` matters here. Without it, `requests` would silently follow the redirect and I'd lose the actual status code that carries the real signal. The moment a `302` comes back, that character is confirmed correct, gets appended to `known`, and the loop breaks out to start guessing the next position from scratch. 

Let it run, and it extracted the flag one confirmed character at a time, fully unattended.
