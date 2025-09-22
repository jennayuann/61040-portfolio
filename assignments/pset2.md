# Pset 2: Composing Concepts

## Concepts for URL Shortening
1. A context is like an identifier for a namespace that allows unqiueness for that concept within that namespace. In the URL Shortening app, a context would be a short URL base domain, such as *tinyurl.com*. That way, the suffix *abc* generated under one doesn't clash with *abc* generated under another.
2. The specification must record which strings have been produced so far so that the precondition of not generating a string already used can be enforced. At the specification level, this is modeled as storing the set of used strings for each context. A concrete implementation of that abstract idea is to maintain a counter that increments everytime generate is called for that context, where the counter value represents all the strings that have been generated so far.
3. One advantage of using common dictionary words as URL suffixes is that it's easy to remember, pronounce, and share for the user compared to random letters or numbers. A disadvantage is that it may be more easily guessed compared to more obscured suffixes and reduce security for the user. To modify *NonceGeneration* concept to realize this idea, we can add a dictionary of *allowed_words* for each context. Then change the *generate* action so that instead of inventing a random string, it picks a word from *allowed_words* that hasn't been used yet in that context. Lastly, track *words_in_use* as a set so no word is repeated within every context.

## Synchonizations for URL Shortening
1. In the sync *generate*, the *Request.shortenUrl* action only binds *shortURLBase* because that's all we need to call *NonceGeneration*.*generate(context: shortURLBase)*, so this sync will match any *Request.shortURL* regardless of *targetURL*. In the sync *register*, we need to bind both *targetURL* and *shortURLBase* in order to call *UrlShortening.register*, so both arguments must appear in the when clause.

2. The omitting names convention only works when the argument name and the variable name are identical. If the names are different, if multiple actions return values with the same name, or if leaving out the name would make the code confusing, then it's best to write out the full mapping for clarity.

3. The *request* action is included in the first two syncs because they are triggered by the user's request and require its inputs to invoke the next actions. The third sync is not triggered by the user but by the completion of *URLShortening.register*, and the original request adds no information since all it needs is the returned *shortURL*.

4. To implement a fixed domain, say *bit.ly* for example, I would replace all instances of the variable *shortURLBase* with the constant *"bit.ly"* in my synchronizations.

5.
```
sync expire
when ExpiringResource.expireResource(): (resource: shortUrl)
then UrlShortening.delete(shortUrl)
```

## Extending the Design

### Concept 1: ResourceOwnership
```
concept ResourceOwnership[User, Resource]
purpose: record who has ownership of each resource
principle: after assigning an owner to a resource, the system can recognize that user as the resource's owner; ownership of a resource can be later be transferred or removed

state
    a set of Resources with
        an owner User

actions
    assignOwner(resource: Resource, user: User)
        requires: owner of *resource* is none
        effect: owner of *resource* is now *user*

    transferOwner(resource: Resource, from: User, to: User)
        requires: owner of *resource* is user *from*
        effect: owner of *resource* becomes user *to*

    removeOwner(resource: Resource, user: User)
        requires: owner of *resource* is *user*
        effect: there is now no owner to *resource*
```

### Concept 2: LinkAnalytics[User, Link]
```
purpose: count accesses to links while keeping analytics private to authorized user
principle: after a link is accessed by a user, its count increments; only authorized user can view the count

state
    a set of Links with
        a count Number
        an admin authorized User // the registrant of the shortening in this case

actions
    recordAccess(link: Link)
        effect: increment viewership count of *link*; if not pre-existing in system, set initialize viewship count of *link* to 1

    removeAccess(link: Link)
        requires: *count* of given *link* must be a number greater than 0
        effect: decrement viewership count of *link*

    viewCount(link: Link, requester: User): (count: Number)
        requires: *requester* to be the *admin* for this *link*
        effect: return *count* of this *link*
```

### Concept 3: UserAuth[User]
```
purpose: identify the user who is currently authenticated
principle: after a user logs in, the system can report who the current user is

state
    a current User (optional)

actions
    login(user: User)
        effect: set *current* user to given *user*

    logout()
        effect: set *current* to none

    getCurrentUser(): (user: User)
        requires: *current* is not none
        effect: return the *current* user
```

### Sync 1: Creating shortenings
```
sync onRegisterSetOwnerAndAdmin

when
    UrlShortening.register(): (shortUrl)
    UserAuth.getCurrentUser(): (user)
then
    ResourceOwnership.assignOwner(shortUrl, user)
    LinkAnalytics.admin(shortUrl) = user
```

### Sync 2: Shortenings translated to targets
```
sync countOnLookup
when UrlShortening.lookup(shortUrl): (targetUrl)
then LinkAnalytics.recordAccess(shortUrl)
```

### Sync 3: User examines analytics
```
sync viewAnalytics

when
    Request.viewAnalytics(shortUrl)
    UserAuth.getCurrentUser(): (user)
then
    LinkAnalytics.viewCount(requester: user, link: shortUrl): (count)
```

### Potential Features:
1. Allowing users to create their own short URLs is a good feature to add. To do this, we can add an optional desiredSuffix to Request.shortenUrl. We would also need to create a new ShortUrlPolicy concept that ensures length, validates characters, disallows profanity, etc. In the generate\register sync, if desiredSuffix is present, invoke ShortUrlPolicy.check(desiredSuffix); if successful, call UrlShortening.register(shortUrlSuffix: desiredSuffix, ...); otherwise, fall back onto NonceGeneration.generate.
2. Using the "word as nonce" strategy can be implemented by keeping UrlShortening untouched but changing out the generator behind the sync. Add a WordNonceGeneration[Context] that would manage words dictionary words and returns unique unused words. Modify the existing generate sync to call WordNonceGeneration.generate(context: shortUrlBase) instead of NonceGeneration.generate.
3. Including the target URL in analytics is an undesirable feature and shouldn't be included because it can allow one creator see traffic that belongs to other people who happen to point at the same target URL. This would be a privacy leak and an easy way for encourage spying on competitors.
4. Generate short URLs that are not easily guessed is a good feature to pad extra security. Add SecureNonceGeneration[Context] concept that generates a less common and less-easily guessable chain of characters or word, and a NoncePolicy[Context] concept that sets rules such as character length. Update the generate sync to call SecureNonceGeneration.generate(context: shortUrlBase). If a user provides their own desired suffix, add a guard sync that calls NoncePolicy.check(suffix) before UrlShortening.register.
5. I believe the feature of allowing unregistered users to view analytics of their own short URLs is an undesirable feature because it could potentially lead to an abuse of power and weaken privacy and accountability. Without an account, the system can't reliably verify identity. This unregistered creator gains access to data while the system has none of their information to hold them accountable if something goes wrong.
