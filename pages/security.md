<!--
 This is the Terms of Service that appears at http://var.ci/privacy/
 You can also find me at http://github.com/varci/support
 -->

# Security

> v1.0.0 - Edit on [GitHub](https://github.com/varci/support/blob/master/pages/security.md)

**TL;DR**

We value your trust in our team. We will honor our commitment to provide an 
awesome & secure product.

#### Contents

1.  [How does VarCI authenticate access to repository?](#how-does-varci-authenticate-access-to-a-repository)
2.  [How does VarCI store passwords?](#how-does-varci-store-passwords)
3.  [How does VarCI store Tokens?](#how-does-varci-store-tokens)
4.  [Does VarCI store source code?](#does-varci-store-source-code)
5.  [How does VarCI archive the bot's logs?](#how-does-varci-archive-the-bots-logs)
6.  [Is my Team's data isolated from other Teams?](#is-my-teams-data-isolated-from-other-teams)
6.  [How do I add collaborators/members to my private repository?](#how-do-i-add-collaboratorsmembers-to-my-private-repository)
7.  [What Scope is used when signing-up using GitHub?](#what-scope-is-used-when-signing-up-using-github)
8.  [Can I restrict which Teams VarCI has access to?](#can-i-restrict-which-teams-varci-has-access-to)
9.  [Can I grant VarCI access only to issues?](#can-i-grant-varci-access-only-to-issues)
10. [Does VarCI ever clone the repository?](#does-varci-ever-clone-the-repository)
11. [When does VarCI read source code from my repository?](#when-does-varci-read-source-code-from-my-repository)
12. [When does VarCI write to my repository?](#when-does-varci-write-to-my-repository)
13. [Are logs kept on who accesses what data on VarCI?](#are-logs-kept-on-who-access-what-data-on-varci)
14. [How do I change the configuration on the repository?](#how-do-i-change-the-configuration-on-the-repository)

#### Terms

-   **VarCI** VarCI and its technology/product/services
-   **Team** a team or organization in GitHub
-   **Repo** a Service (public or private) repository
-   **User** a single person who has logged into VarCI via GitHub therefore has 
    an active user session
-   **Guest** a http request performed without an active user sessions
-   **Worker** VarCI's sync back-end which handles uploading, report processing, 
    and other tasks
-   **Bot** the User who was chosen to consume GitHub endpoints during Worker 
    tasks
-   **Web** VarCI front-end service that handles page builds and all HTTP 
    requests (GET, POST, etc.)
-   **Token** a Users auth token/secret granted by GitHub upon logging-in to 
    VarCI
-   **Scope** what level of permission a User has granted VarCI on GitHub
-   **CI** continuous integration provider. Including (not limited to) Travis-CI, 
    Circle CI, Jenkins, etc.
-   **API** HTTP requests to GitHub
-   **3rd Party** a SaaS tool used by VarCI. *Example* [Logz.io](https://logz.io)

--

>   NOTE: We've decided to save time and get inspired by the documents used by
>   great service providers like [Codecov](https://codecov.io). They've been 
>   there, done that; we piggy-backed. 

--

## How does VarCI authenticate access to a repository?

-   **Public** Repos are visible to all Users and Guests

-   **Private** Repos are visible to Users who have at least `read` access 
    according to Service

-   VarCI checks the User's Scope by making an API request with the User's 
    Token

-   If the User **does not** have at least `read` access to the Repo: VarCI 
    will return a 404 HTTP Error

-   VarCI **always** uses the acting User's Token to make API requests to
    Service *when navigating VarCI*

-   VarCI **always** uses the Bot's Token in when preforming Worker tasks

## How does VarCI store passwords?

-   VarCI **does not** use passwords in the product.

-   VarCI does not, **ever**, ask for any "passwords".

-   VarCI stores Tokens for Users upon logging-in.

## How does VarCI store Tokens?

-   VarCI receives Tokens when a User logs into VarCI.

-   Tokens are **encrypted** by [AES](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard).
    The key used to encrypt Tokens is broken into three chunks stored in 
    different locations in the Stack to reduce a single point of failure. In 
    order to compromise Tokens an attacker must breach multiple levels of the 
    VarCI Stack (database, source code, server environment, and more).

-   Only VarCI staff have access to User Tokens.

-   Tokens are aggressively removed from any logs and tracebacks and are never 
    sent to 3rd Party solutions.

## Does VarCI store source code?

> TL;DR We NEVER read the source code unless you instruct us to.

There is never a need for VarCI to read the source code when dealing with 
issues and pull requests. But given that GitHub only allows users with push 
access to label/close comments, VarCI requires the `write:org` scope.

However, in some cases where you are trying to enforce a rule on the content
of a pull request (i.e. the awesome list's [workflow]), all of the pull 
request's content (`diff`) is downloaded and only the files you instructed us 
to protect are parsed.

## How does VarCI archive the bot's logs?

VarCI archives both, the incoming webhook payload and the subsequently triggered
API payloads in AWS S3.

The logs are kept private to VarCI for now but we are looking into making them
available to authorized users; to avoid vendor lock-in and verify accuracy.

## Is my Team's data isolated from other Teams?

No, Teams/Repos/Users data is stored in one or more databases, all property of
VarCI, but not isolated from other Teams/Repos/Users utilizing VarCI services.

## How do I add collaborators/members to my private repository?

-   A User's access is always verified with GitHub.

-   If using GitHub: once a User logs in they must grant VarCI Private 
    Repository Access in order to interact with Private repositories hosted on
    GitHub that User has appropriate access to.

This allows us to have 100% transparency on who can access source code and view 
reports on VarCI.

## What Scope is used when signing-up using GitHub?

VarCI first asks for `user:email, read:org, repo, write:repo_hook` Scope,
**note** this is for **Public Repos only**. Once a User is logged-in they may
elect to grant VarCI extended privileges which enable the User to interact with
**Private Repos**.

> Read more on [GitHub Scopes here](https://developer.github.com/v3/oauth/#scopes)

## Can I restrict which Teams VarCI has access to?

No. This is currently a limitation to GitHub. This feature would have to be
implemented  by GitHub; not by VarCI.

## Can I grant VarCI access only to issues?

No. This is currently a limitation to GitHub. This feature would have to be
implemented  by GitHub; not by VarCI.

## Does VarCI ever clone the repository?

No, never. VarCI uses API requests to retrieve information necessary to perform
its job and never deals with source code (not even at the API level).

## When does VarCI read source code from my repository?

Never.

## When does VarCI write to my repository?

The only times VarCI will "write" to your repository is in the following
processes:

1.  Create/Update a Webhook
2.  Create/Update/Delete an Issue comment
2.  Create/Update/Delete a Pull Request Comment
3.  Create/Update the Commit Status

>   VarCI never adjusts source code, deletes branches, issues or pull requests, 
>   or performs any other 'write' action.

## Are logs kept on who accesses what data on VarCI?

Yes. Each and every Web request is logged for a period of one year or longer.
Logs are accessible by VarCI staff and are used to analyze User behavior and
help debug the product.

## How do I change the configuration on the repository?

Most Repo configuration is recorded in a file called `varci.yml` within the
Repo. The  location of this configuration file may be anywhere within the Repo
and must be named  `varci.yml` or `.varci.yml` in order to be detected. Having
configuration stored in the  `varci.yml` allows for complete transparency and
version controlled configuration.

For more details please visit https://var.ci/configuration
