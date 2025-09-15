# Problem Set 1

## **Exercise 1:**
1. The first invariant is that for every item in a registry, the number still requested must equal the original number requested minus the total number of purchases that have been made for that item, and this number cannot go below zero. The second invariant is that every purchase must correspond to an item that was requested in the registry. The second invariant is more important because it ensures that every purchase is properly tied to a request. This protects the core motivation behind this concept that friends are only buying gifts that you've requested. The *purchase* action is most affected because of its requirement that the request must exist which preserves this invariant.
2. *removeItem* might break this invariant. If an item has already been purchased and then the request is removed, all purchases would no longer have a corresponding request, which would violate the invariant. To fix this, once a purchase for a request exists, the system should either disallow removals or archive the request in a way that can maintain a proper purchase history to preserve the invariant.
3. The action specifications allow a registry to be opened and closed repeatedly since the *open* action fulfills the requirements for the *close* action and the *close* action fulfills the requirements for the *open* action. This flexibility is useful for users who want to temporarily close their registry while editing or for users who only need it open during certain hours of the day.
4. In practice, having no way to delete a registry should be fine because the user can just close that registry and create another one if needed. Having old registries not deleted could come in useful for record keeping, such as handling returns and other disputes.
5. For the registry owner, a common query is "Which items were purchased, how many, by whom, and how many are still unpurchased?". For a giver, a common query is "Which items are available to purchase and how many of each?".
6. To support the surprise factor, we can add a flag that the registry owner can turn on that hides purchase details from them until the registry is closed. While this flag is active, the owner would only see the details of what they initially put on the registry such as the items and their count numbers. Once the flag is turned off or the registry is closed, the full purchase details become visible.
7. Leaving User and Items as generic types makes the registry more modular and flexible. This allows the concept to be connected to many different systems without being constrained to fixed representations of users or items. For example, items can be represented by something more stable like SKU codes or product IDs rather than names, descriptions, or prices which may change over time and cause inconsistencies in the system. This separation allows the registry to handle how requests and purchases work while leaving details like item and user data to the external system.

## **Exercise 2:**
1. **state**
- a set of Users with
    - a username String
    - a password String
    - an email String
    - confirmed Flag
    - secretToken string // set to undefined after email confirmation
2. **actions**
- register (username: String, password: String, email: String): (user: User, token: String)
    - requires: no User exists with this username
    - effects: creates a new user with this given username, password, and email; confirmed is set to False and secretToken is set to a fresh new token; return user and the fresh token.
- authenticate (username: String, password: String): (user: User)
    - requires: a User exists with this username and password and the user is confirmed.
    - effects: no state change; return user.
- confirm (username: String, secretToken: String)
    - requires: User exists with this username and secretToken and user is currently unconfirmed.
    - effects: set this User's confirmed = True and secretToken = undefined.
3. The most important invariant is that a user can only authenticate if the given password is equal to the stored password for that username. It's preserved by both the *register* and *authenticate* actions. *register* first establishes the mapping between usernames and passwords when a new user is created, and *authenticate* only runs to completion if the invariant holds, which is that the username and password are both correct.
4. To implement this extension, I added a email string, confirmed flag, and secretToken string to the state. Then the *register* action now takes in an email, sets confirmed to false, and sets secretToken to a fresh secret token. I also added a *confirm* action that takes in a username and secret token and if valid, sets the confirmed to true and secretToken to undefined to indicate that email confirmation has been completed.

## **Exercise 3:**
- **concept** PersonalAccessToken[User, Scope]
- **purpose:** authenticate a user in place of their password, especially for command-line, API, and automation access.
- **principle:** a user generates a token with chosen scope and expiration; the token is given once and must be securely stored by the user; the user can later present this token to authenticate in place of their password; if the token is still active and not expired, the server will grant access according to its scope; the user can delete the token at any time.
- **state**
    - a set of Tokens with
        - an owner User
        - an accessToken String
        - a set of Scopes
        - an expiration Time? (optional)
- **actions**
    - generateToken(owner: User, scopes: set of Scopes, expiration: Time?): (token: Token, accessToken: String)
        - requires: owner exists
        - effects: create a new token with given owner, scopes, and expiration with *accessToken* set to a fresh random string; both the new *token* object and *accessToken* string are returned.
    - authenticateWithToken(accessToken: String): (user: User)
        - requires: a non-expired *Token* exists with given *accessToken*.
        - effects: no state change, user is authenticated as owner.
    - deleteToken(token: Token)
        - requires: given token exists.
        - effects: remove the given token from the set of Tokens.
- **How PersonalAccessToken differs from PasswordAuthentication:**
    - Users can generate multiple tokens each with its own scopes and optional expiration, whereas passwords are limited to one and allows full-access.
    - Tokens can be deleted independent of one another, whereas password resets affect the whole account.
    - Tokens are more intended for scripts and command-line use, whereas passwords are mainly for signing in through the website or other interactive logins.
- **How to change GitHub documentation:** The documentation currently says "treat your access tokens like passwords," which blurs their differences. Instead, GitHub should clearly explain their differences. An idea to do so is through a table with one column for a password and another for tokens: a password is one per user, allows full access, and interactive; whereas tokens are many per user, scoped, expiring, deletable, and more suited for command line and scripts.

## **Exercise 4:**

### URL Shortener
- **concept** URLShortener[User]
- **purpose:** map long URLs to shorter URLs to simplify the process of sharing and visiting websites.
- **principle:** a user submits a URL and optionally a suffix and domain; if the suffix and domain pair is unused, the system creates a short URL with that domain and suffix; otherwise the system generates a random unused suffix (on the given or default domain); anyone can later use the generated short URL to be redirected to the original URL.
- **state**
    - a set of ShortURLs with
        - owner User? (optional)
        - longURL String
        - domain String (service default if none given)
        - suffix String (unique per domain)
- **actions**
    - generateShortURL(owner: User?, longURL: String, suffix: String?, domain: String?): (shortLink: String)
        - requires: if given a *suffix*, no *ShortURL* exists with the same *domain* and *suffix* pair.
        - effects: generate a new *ShortURL* with:
            - domain = domain if given, otherwise defaults to the service's domain
            - suffix = suffix if given, otherwise a random unused suffix for that domain
            - longURL = longURL
            - owner = owner if provided, otherwise owner = undefined
            - Return the concatenated short link string: *domain* + '/' + *suffix*.
    - resolveShortURL(domain: String, suffix: String): (longURL: String)
        - requires: a ShortURL exists with this given domain and suffix pair.
        - effects: no state change; return the associated longURL.

### Billable Hours Tracking
- **concept** BillableHours[Project, Employee]
- **purpose:** record the time employees spend working on projects for billing purposes to avoid disputes.
- **principle:** an employee starts a work session by selecting a project and entering a description describing the work to be done; the system records the start time; the employee later ends the session, which sets the end time; if the employee forgets to end the session, they can end it later and give a corrected end time.
- **state**
    - a set of Sessions with
        - an employee Employee
        - a project Project
        - description String
        - startTime Time
        - endTime Time? (undefined until ended)
- **actions**
    - startSession(employee: Employee, project: Project, description: String): (session: Session)
        - requires: employee exists, project exists, and employee has no active session.
        - effects: create a new Session with given employee, project, description,  startTime = now, and endTime = undefined; return the session.
    - endSession(session: Session, endTime: Time?)
        - requires: given session exists, current endTime = undefined, and if given an endTime, it must be after the session's startTime.
        - effects: set given session's endTime = now if not given an endTime, otherwise set endTime = endTime.

### Conference Room Booking
- **concept** RoomReservation[User, Room]
- **purpose:** allocate rooms to users to avoid scheduling conflicts.
- **principle:** rooms are registered with their operating hours and capacity; a user requests a reservation for a specific room and time interval; if the room has no overlapping reservations in that interval and the interval is within its operating hours, the reservation is created; the user may later modify the time or cancel the reservation; anyone is able to check the availability of a room; operating hours and capacities of rooms may be modified.
- **state**
    - a set of Rooms with
        - room Room
        - operatingHours (Time, Time)? (if not filled, room is always bookable)
        - capacity Number?
    - a set of Reservations with
        - room Room
        - organizer User
        - title String
        - startTime Time
        - endTime Time
- **actions**
    - registerRoom(room: Room, operatingHours: (Time, Time)?, capacity: Number?)
        - requires: no Room entry exists for *room*.
        - effects: add {*room*, *operatingHours*, *capacity*} to Rooms; unspecified fields are left undefined.
    - setOperatingHours(room: Room, operatingHours: (Time, Time)?)
        - requires: a Room entry exists for room and every existing Reservation for *room* must be within these new hours.
        - effects: update the room's operating hours; if operatingHours not provided, clear room's operatingHours by setting it to undefined.
    - setCapacity(room: Room, capacity: Number?)
        - requires: a Room entry exists for *room*.
        - effects: update the room's capacity; if capacity not provided, clear room's capacity by setting capacity to undefined.
        - Note: this concept does not track attendee counts, capacity is just data for the organizer to track.
    - createReservation(organizer: User, room: Room, title: String, startTime: Time, endTime: Time): (reservation: Reservation)
        - requires: *organizer* exists, *room* exists, *startTime* to be before *endTime*, no Reservation exists with this *room* that overlaps this time interval, and time interval is within room's operating hours if defined.
        - effects: create a new Reservation with the given user, room, title, start time, and end time; return the created *reservation*.
    - cancelReservation(requester: User, reservation: Reservation)
        - requires: *reservation* exists and the *requester* is the *organizer* of the reservation.
        - effects: remove *reservation*.
    - modifyReservationTime(requester: User, reservation: Reservation, newStartTime: Time, newEndTime: Time)
        - requires: *reservation* exists, the *requester* is the *organizer* of the reservation, *newStartTime* to be before *newEndTime*, no Reservation (other than original *reservation*) overlaps with this new time interval in the room, and time interval is within room's operating hours if defined.
        - effects: set reservation's *startTime* to *newStartTime* and *endTime* to *newEndTime*.
    - isAvailable(room: Room, startTime: Time, endTime: Time): (available: Flag)
        - requires: *room* exists and *startTime* must be before *endTime*.
        - effects: no state change; returns true if no Reservation exists with this *room* that overlaps the provided time interval and the time interval is within the room's operating hours, otherwise return false.
- Overlap definition: Two time intervals overlap if they cover any of the same time. They do not overlap if one ends exactly when the other begins (start time is inclusive and end time is exclusive).
