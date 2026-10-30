# Session 5 & 6 — Exam Review Notes
Compiled from our walkthrough: concepts + the code we built together (Kigali Weather guide, Firebase/Firestore guide, and your Study Room Booking System applied across both). This is a study reference, not a project to hand in — you wrote the Booking/UsageRecord/security-rule pieces yourself; make sure you can still explain every line below out loud.

**Session 5** (async/await, REST APIs, `shared_preferences`) is below. **Session 6** (Firebase: Auth, Firestore, Streams, security rules) starts at §9.

---

## 1. Future, async, await

A `Future<T>` is "a `T`, eventually" — the *ticket*, not the food.
`async` marks a function as allowed to pause. `await` is *where* it pauses — only that function waits; nothing else freezes.

```dart
Future<String> orderBrochette() async {
  await Future.delayed(Duration(seconds: 3));
  return 'Your brochette is ready';
}
```

Forgetting `await` gets you `Instance of 'Future'` — the ticket, not the food.

---

## 2. Calling an API + turning JSON into a Dart object

```dart
class Weather {
  final double temperature;
  final int humidity;
  final double wind;
  final int code;

  Weather({required this.temperature, required this.humidity, required this.wind, required this.code});

  factory Weather.fromJson(Map<String, dynamic> json) => Weather(
    temperature: (json['temperature_2m'] as num).toDouble(),
    humidity: (json['relative_humidity_2m'] as num).toInt(),
    wind: (json['wind_speed_10m'] as num).toDouble(),
    code: (json['weather_code'] as num).toInt(),
  );
}

Future<Weather> fetchWeather(City city) async {
  final url = Uri.parse(
    'https://api.open-meteo.com/v1/forecast'
    '?latitude=${city.latitude}&longitude=${city.longitude}'
    '&current=temperature_2m,relative_humidity_2m,wind_speed_10m,weather_code',
  );
  final response = await http.get(url);
  if (response.statusCode != 200) {
    throw Exception('The weather server answered ${response.statusCode}');
  }
  final data = jsonDecode(response.body) as Map<String, dynamic>;
  return Weather.fromJson(data['current'] as Map<String, dynamic>);
}
```

**Why `as num` then `.toDouble()`/`.toInt()`, only for external API data?**
JSON doesn't distinguish `25` from `25.0`. The *server's* number might arrive either way depending on the actual value, so read as `num` (either kind) then convert. This does **not** apply to data only your own `toJson` ever writes (e.g. `numberOfOccupants`) — that always round-trips as the same type, so a plain `as int` is correct and *more* honest there.

**Why check `statusCode` before parsing the body?**
A non-200 response might not even be JSON (could be an HTML error page). Checking first lets you `throw` your own clear exception instead of getting a confusing `FormatException` from `jsonDecode`.

**Why does one `catch` block handle "bad status" *and* "no internet at all"?**
With no internet, `http.get()` itself throws (a `SocketException`) before any response exists — execution jumps straight from `try` into `catch`, skipping the status check entirely. One generic `catch (e)` covers both failure modes with the same Retry UI.

---

## 3. The three-state UI pattern (loading / error / data)

```dart
bool loading = true;
String? error;
Weather? weather;

Future<void> load() async {
  setState(() { loading = true; error = null; });
  try {
    final w = await fetchWeather(selectedCity);
    setState(() { weather = w; loading = false; });
  } catch (e) {
    setState(() { error = 'Could not load the weather. Check your internet connection.'; loading = false; });
  }
}

// build(): if (loading) show spinner; else if (error != null) show message + Retry; else show data.
```

`build()` never fetches — it only *reads* state and decides which of the three to show. Fetching is a side effect, so it only happens explicitly: `initState()`, `onSelected`, and the Retry button's `onPressed`. `build()` can re-run many times for unrelated reasons (animations, parent rebuilds), so it must stay side-effect-free.

---

## 4. Switching between options (cities)

```dart
class City {
  final String name;
  final double latitude;
  final double longitude;
  City(this.name, this.latitude, this.longitude);
}

final cities = [
  City('Kigali', -1.9441, 30.0619),
  City('Huye', -2.5967, 29.7394),
  City('Musanze', -1.4998, 29.6344),
  City('Rubavu', -1.6792, 29.2585),
];
// ChoiceChip onSelected: setState(() => selectedCity = city); load();
```

Updating `selectedCity` only triggers a rebuild — it does not fetch. `onSelected` must call `load()` itself.

---

## 5. Remembering a single choice (`shared_preferences`)

```dart
final prefs = await SharedPreferences.getInstance();   // itself async — touches disk
await prefs.setString('city', selectedCity.name);      // save AFTER fetch succeeds
// ...
final saved = prefs.getString('city');                 // 'Musanze', or null the first time
```

`shared_preferences` only stores `String`/`int`/`bool`/`double`/`List<String>` — never a whole object.

Save **after** success, not on tap: if you saved immediately and the fetch failed, next launch would try to restore a city you never actually confirmed worked — and (see §6) there'd be no cached weather for it either.

---

## 6. Caching one object for offline use (`toJson` / `jsonEncode`)

```dart
await prefs.setString('weather_${selectedCity.name}', jsonEncode(weather.toJson()));
// on failure:
final cached = prefs.getString('weather_${selectedCity.name}');
if (cached != null) weather = Weather.fromJson(jsonDecode(cached));
```

Chain to memorize: object → `toJson()` → `Map` → `jsonEncode` → `String` (to store). Reverse to load: `String` → `jsonDecode` → `Map` → `fromJson()` → object.

---

## 7. Saving/loading a *list* of your own objects (Part 2 generalization)

Your `Booking` and `UsageRecord` models (Study Room Booking System):

```dart
enum BookingStatus { requested, approved, used, cancelled }

class Booking {
  final String id;
  final String studentName;
  final String supervisorName;
  final DateTime date;
  final int startingHour;
  final String purpose;
  final BookingStatus status;

  Booking({required this.id, required this.studentName, required this.supervisorName,
    required this.date, required this.startingHour, required this.purpose, required this.status});

  Map<String, dynamic> toJson() => {
    'id': id, 'studentName': studentName, 'supervisorName': supervisorName,
    'date': date.toIso8601String(), 'startingHour': startingHour,
    'purpose': purpose, 'status': status.name,
  };

  factory Booking.fromJson(Map<String, dynamic> json) => Booking(
    id: json['id'] as String,
    studentName: json['studentName'] as String,
    supervisorName: json['supervisorName'] as String,
    date: DateTime.parse(json['date'] as String),
    startingHour: json['startingHour'] as int,
    purpose: json['purpose'] as String,
    status: BookingStatus.values.byName(json['status'] as String),
  );

  Booking copyWith({String? id, String? studentName, String? supervisorName, DateTime? date,
      int? startingHour, String? purpose, BookingStatus? status}) => Booking(
    id: id ?? this.id,
    studentName: studentName ?? this.studentName,
    supervisorName: supervisorName ?? this.supervisorName,
    date: date ?? this.date,
    startingHour: startingHour ?? this.startingHour,
    purpose: purpose ?? this.purpose,
    status: status ?? this.status,
  );
}

class UsageRecord {
  final String id;
  final String bookingId;                 // links back to its Booking
  final List<String> equipmentUsed;
  final String remarks;
  final int numberOfOccupants;
  final DateTime recordedDate;

  UsageRecord({required this.id, required this.bookingId, required this.equipmentUsed,
    required this.remarks, required this.numberOfOccupants, required this.recordedDate});

  Map<String, dynamic> toJson() => {
    'id': id, 'bookingId': bookingId, 'equipmentUsed': equipmentUsed,
    'remarks': remarks, 'numberOfOccupants': numberOfOccupants,
    'recordedDate': recordedDate.toIso8601String(),
  };

  factory UsageRecord.fromJson(Map<String, dynamic> json) => UsageRecord(
    id: json['id'] as String,
    bookingId: json['bookingId'] as String,
    equipmentUsed: List<String>.from(json['equipmentUsed'] ?? []),
    remarks: json['remarks'] as String? ?? '',
    numberOfOccupants: json['numberOfOccupants'] as int,
    recordedDate: DateTime.parse(json['recordedDate'] as String),
  );
}
```

Enums aren't JSON types either: store `status.name` (a `String`), read back with `BookingStatus.values.byName(...)`.

**Saving/loading the whole list** — the step-8 pattern generalized:

```dart
const _bookingsKey = 'bookings_key';

Future<void> saveBookings(List<Booking> bookings) async {
  final prefs = await SharedPreferences.getInstance();
  await prefs.setString(_bookingsKey, jsonEncode(bookings.map((b) => b.toJson()).toList()));
}

Future<List<Booking>> loadBookings() async {
  final prefs = await SharedPreferences.getInstance();
  final saved = prefs.getString(_bookingsKey);
  if (saved == null) return [];
  final jsonList = jsonDecode(saved) as List;
  return jsonList.map((j) => Booking.fromJson(j as Map<String, dynamic>)).toList();
}
```

The chain: `List<Booking>` → `.map(toJson)` → `List<Map>` → `jsonEncode` → one `String`. Reversed exactly to load. Same four steps for *any* list of *any* model — `saveUsageRecords`/`loadUsageRecords` are identical with the model and key string swapped.

**Cast explicitly at the boundary**: write `Model.fromJson(j as Map<String, dynamic>)`, not `Model.fromJson(j)` — `j` is `dynamic` either way, but the explicit cast fails immediately with a clear error if an item is ever malformed, instead of failing confusingly later inside `fromJson`.

**Linking two lists**: filter by the foreign key, don't duplicate data:

```dart
List<UsageRecord> usageFor(String bookingId, List<UsageRecord> all) =>
    all.where((r) => r.bookingId == bookingId).toList();
```

---

## 8. Updating one item inside a saved list

Two valid approaches:

```dart
// A) Functional — rebuild the whole list
Future<void> markBookingAsUsed(String bookingId) async {
  final bookings = await loadBookings();
  final updated = bookings.map((b) => b.id == bookingId ? b.copyWith(status: BookingStatus.used) : b).toList();
  await saveBookings(updated);
}

// B) Imperative — mutate one slot
final index = bookings.indexWhere((b) => b.id == bookingId);
if (index != -1) bookings[index] = bookings[index].copyWith(status: BookingStatus.used);
```

Both are correct. (A) rebuilds the list; (B) mutates one slot and needs an explicit "not found" guard (`-1`) that (A) doesn't, since `.map` silently no-ops on no match.

---
---

# Session 6 — Firebase (Auth, Firestore, Streams, Security Rules)

## 9. Future vs Stream

A `Future` is a box: filled once, then done (logging in, saving one document) — pairs with `await` / `FutureBuilder`.
A `Stream` is a pipe: delivers a value now, and *again* every time something changes (who's logged in, the live list of data) — pairs with `.listen()` / `StreamBuilder`.

Rule of thumb: "do something" → `Future`. "Watch something that can change while I'm looking at it" → `Stream`. Login state is a `Stream` (`authStateChanges()`) even though it feels like a one-time check, because it *can* change while the app is open (sign out, token expiry).

```dart
Future<void> login(String email, String password) async {
  await FirebaseAuth.instance.signInWithEmailAndPassword(email: email, password: password);
}

Stream<User?> whoIsLoggedIn = FirebaseAuth.instance.authStateChanges();
```

---

## 10. Auth + specific exception types

```dart
Future<void> signUp(String email, String password) async {
  try {
    await FirebaseAuth.instance.createUserWithEmailAndPassword(email: email, password: password);
  } on FirebaseAuthException catch (e) {
    setState(() { error = _message(e.code); });
  }
}

String _message(String code) {
  switch (code) {
    case 'email-already-in-use': return 'An account already exists for that email.';
    case 'weak-password': return 'Choose a stronger password (6+ characters).';
    case 'invalid-credential':
    case 'wrong-password':
    case 'user-not-found': return 'Email or password is incorrect.';
    case 'network-request-failed': return 'No internet connection.';
    default: return 'Something went wrong. Try again.';
  }
}
```

**Why `on FirebaseAuthException catch (e)` instead of a generic `catch (e)`?** It lets you react to specific error codes with specific messages — *and*, just as important, anything that **isn't** a `FirebaseAuthException` (a bug in your own code) is **not** caught here. It propagates instead, so you see it and fix it, rather than it being silently swallowed into a generic "something went wrong."

---

## 11. Saving data: `toMap()` / `fromDoc()`, and why the server's clock

```dart
class Txn {
  final String? id;
  final String description;
  final double amount;
  final bool isIn;
  final DateTime? createdAt;

  Txn({this.id, required this.description, required this.amount, required this.isIn, this.createdAt});

  Map<String, dynamic> toMap() => {
    'description': description,
    'amount': amount,
    'isIn': isIn,
    'createdAt': FieldValue.serverTimestamp(),
  };

  factory Txn.fromDoc(DocumentSnapshot<Map<String, dynamic>> doc) {
    final data = doc.data()!;
    return Txn(
      id: doc.id,
      description: data['description'] as String,
      amount: (data['amount'] as num).toDouble(),
      isIn: data['isIn'] as bool,
      createdAt: (data['createdAt'] as Timestamp?)?.toDate(),
    );
  }
}

CollectionReference<Map<String, dynamic>> myTransactions() {
  final uid = FirebaseAuth.instance.currentUser!.uid;
  return FirebaseFirestore.instance.collection('users').doc(uid).collection('transactions');
}

Future<void> addTransaction(Txn txn) async => await myTransactions().add(txn.toMap());
```

**Why `FieldValue.serverTimestamp()` instead of `DateTime.now()`?** Phones can't be trusted — clocks can be wrong, time zones differ, and a user could spoof their own device clock to make a transaction look older/newer than it is. The server decides, not the device. Same principle as security rules running server-side instead of trusting an `if` in Flutter.

Note: this applies to *audit* timestamps (`createdAt` — "when did this write happen"). A field like a chosen booking date/time is a different kind of fact — the user picks it, so it's stored as a real `DateTime`/`Timestamp.fromDate(...)`, not `serverTimestamp()`.

---

## 12. `StreamBuilder`, four states, and "save facts, compute the rest"

```dart
StreamBuilder<QuerySnapshot<Map<String, dynamic>>>(
  stream: myTransactions().orderBy('createdAt').snapshots(),
  builder: (context, snapshot) {
    if (snapshot.connectionState == ConnectionState.waiting) {
      return Center(child: CircularProgressIndicator());               // loading
    }
    if (snapshot.hasError) {
      return Center(child: Text('Could not load your transactions.'));  // error
    }
    final docs = snapshot.data!.docs;
    if (docs.isEmpty) {
      return Center(child: Text('No transactions yet. Add one below.')); // empty
    }
    final txns = docs.map((d) => Txn.fromDoc(d)).toList();
    final balance = txns.fold<double>(0, (sum, t) => sum + (t.isIn ? t.amount : -t.amount));
    return Column(children: [
      Text('${balance.toStringAsFixed(0)} RWF'),
      Expanded(child: ListView(children: txns.map((t) =>
        ListTile(title: Text(t.description), trailing: Text('${t.isIn ? '+' : '-'}${t.amount}'))).toList())),
    ]);  // data
  },
)
```

`balance` is never written to Firestore — it's recomputed from the full transaction list every time the stream delivers. **Why never save it?** A saved total is a second copy of the truth; if a write ever fails or a transaction gets deleted, a saved balance can silently disagree with the actual transactions. A computed value *can't* disagree — it *is* the transactions.

**Trace a delete, end to end:** delete removes the doc from Firestore → Firestore pushes a new snapshot down the stream → `StreamBuilder` rebuilds automatically (no manual `setState`) → `fold` reruns on the new (shorter) list → the new balance renders. This full chain — "stop manually tracking when to refresh" — is the real upgrade Streams give you over Session 5's `setState`-driven `load()`.

```dart
Future<void> updateTransaction(Txn txn) async => await myTransactions().doc(txn.id).update(txn.toMap());
Future<void> deleteTransaction(String id) async => await myTransactions().doc(id).delete();
```

---

## 13. Security rules — the server says no

The phone can't be trusted: an `if` in Flutter is a sign, not a lock — anyone can skip your app and call Firestore directly. Rules run **on Google's servers**, where no user can bypass them.

**Console options when creating the database:**
- **Test mode (never use)**: `allow read, write: if true` — anyone, no login required, can read/write everyone's data. Also auto-locks the whole database after 30 days.
- **Production mode (always choose)**: starts fully locked — nothing is readable/writable until you publish your own rule.

**Simple case — data owned by one user** (`users/{uid}/transactions`):
```
match /users/{userId}/{document=**} {
  allow read, write: if request.auth != null && request.auth.uid == userId;
}
```
Both conditions are required together: `request.auth != null` blocks anonymous strangers; `request.auth.uid == userId` blocks *other logged-in users* from reading each other's data by editing the UID in a request. Either one alone leaves a real hole.

**Harder case — data shared between two roles** (e.g. a booking visible to both the student who made it and the supervisor assigned to it): don't nest under one owner's path — use a top-level collection with both ids as fields, and check the relationship instead of the path:

```
match /bookings/{bookingId} {
  allow read: if request.auth != null &&
    (request.auth.uid == resource.data.studentId ||
     request.auth.uid == resource.data.supervisorId);

  allow create: if request.auth != null &&
    request.auth.uid == request.resource.data.studentId;
}
```

**The single most common Firestore rules mistake:** `resource.data` is the *existing* document — use it for `read`/`update`/`delete` checks. `request.resource.data` is the *incoming* data being written — use it for `create`/`update` checks, since on `create` there's no existing document yet for `resource` to refer to. Mixing these two up is the classic bug.
