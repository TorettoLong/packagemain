### Getting started with OAuth2 in Go

Authentication usually is very important part in any application. You can always implement your own authentication system, but it will require a lot of work, registration, forgot password form, etc. That's why OAuth2 was created, to allow user to log in using one of the many accounts user already has.

In this video we'll create a simple web page with Google login using oauth2 Go package.
### Google Project, OAuth2 keys

First of all, let's create our Google OAuth2 keys.

 - Go to [Google Cloud Platform](https://console.developers.google.com/)
 - Create new project or use an existing one
 - Go to Credentials
 - Click "Create credentials"
 - Choose "OAuth client ID"
 - Add authorized redirect URL, in our case it will be `localhost:8080/callback`
 - Get client id and client secret
 - Save it in a safe place

### Structure

We'll do everything in 1 main.go file, and register 3 URL handlers:
 - /
 - /login
 - /callback

### Initial handlers and OAuth2 config

```
go get golang.org/x/oauth2
```

We save google client id and secret in env variables and only use os.Getenv in the code.

```
package main

import (
	"net/http"
	"os"

	"golang.org/x/oauth2"
	"golang.org/x/oauth2/google"
)

var (
	googleOauthConfig = &oauth2.Config{
		RedirectURL:  "http://localhost:8080/callback",
		ClientID:     os.Getenv("GOOGLE_CLIENT_ID"),
		ClientSecret: os.Getenv("GOOGLE_CLIENT_SECRET"),
		Scopes:       []string{"https://www.googleapis.com/auth/userinfo.email"},
		Endpoint:     google.Endpoint,
	}
)

func main() {
	//http.HandleFunc("/", handleMain)
	//http.HandleFunc("/login", handleGoogleLogin)
	//http.HandleFunc("/callback", handleGoogleCallback)
	http.ListenAndServe(":8080", nil)
}
```

### /

Now let's render an HTML on index page

```
const htmlIndex = `<html><body><a href="/login">Google Log In</a></body></html>`

func handleMain(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, htmlIndex)
}
```

### /login

We send random state string. In our cause it's not random.

```
// TODO: randomize it
oauthStateString = "random"

func handleGoogleLogin(w http.ResponseWriter, r *http.Request) {
	url := googleOauthConfig.AuthCodeURL(oauthStateString)
	http.Redirect(w, r, url, http.StatusTemporaryRedirect)
}
```

### /callback

1. check state
2. use `code` to get token
3. use token to get user info

```
func handleGoogleCallback(w http.ResponseWriter, r *http.Request) {
	if r.FormValue("state") != oauthStateString {
		fmt.Println("invalid oauth state")
		http.Redirect(w, r, "/", http.StatusTemporaryRedirect)
		return
	}

	token, err := googleOauthConfig.Exchange(oauth2.NoContext, r.FormValue("code"))
	if err != nil {
		fmt.Printf("code exchange failed: %s\n", err.Error())
		http.Redirect(w, r, "/", http.StatusTemporaryRedirect)
		return
	}

	response, err := http.Get("https://www.googleapis.com/oauth2/v2/userinfo?access_token=" + token.AccessToken)
	if err != nil {
		fmt.Printf("failed getting user info: %s\n", err.Error())
		http.Redirect(w, r, "/", http.StatusTemporaryRedirect)
		return
	}

	defer response.Body.Close()
	contents, err := ioutil.ReadAll(response.Body)
	if err != nil {
		fmt.Printf("failed reading response body: %s\n", err.Error())
		http.Redirect(w, r, "/", http.StatusTemporaryRedirect)
		return
	}

	fmt.Fprintf(w, "Content: %s\n", contents)
}
```

### Test it

```
go run main.go
```

### Conclusion

That's all what we need to do to integrate OAath2 with Google in Go. As you can see, it's only 70 lines of code.

Also, there is a nice package https://github.com/markbates/goth from Mark Bates which provides multi-provider authentications.