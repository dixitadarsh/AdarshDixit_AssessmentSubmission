# AdarshDixit_AssessmentSubmission

# 🏨 Hotel Room Reservation System

A dynamic hotel room reservation system built as part of the Unstop SDE assessment. The system intelligently assigns rooms by minimizing total travel time across floors using a combinatorial optimization algorithm.

---

## 🔗 Links

| Resource | Link |
|----------|------|
| 🌐 Live App | https://adarshdixitassessmentsubmission.vercel.app/ |
| 💻 Repository | https://github.com/dixitadarsh/AdarshDixit_AssessmentSubmission |
| 📄 Solution Doc | _This file_ |

---

## 📋 Problem Summary

A hotel has **97 rooms** across **10 floors**:
- **Floors 1–9**: 10 rooms each (Room 101–110, 201–210, … 901–910)
- **Floor 10**: 7 rooms (Room 1001–1007)
- **Lift/Staircase** is on the left side of the building

Travel time rules:
- Moving **1 room horizontally** = **1 minute**
- Moving **1 floor vertically** = **2 minutes**

Booking rules:
- Max **5 rooms per booking**
- **Same floor preferred** first
- Otherwise, find the **combination that minimizes total travel time**

---

## 🧠 Algorithm — Step-by-Step

### Step 1: Data Modelling

Each room is represented as a JavaScript object with the following properties:

```js
{ id: 101, floor: 1, position: 1, booked: false }
```

- `floor` — which floor the room is on (1–10)
- `position` — how far it is from the lift (1 = closest, 10 = farthest)
- `booked` — whether the room is already occupied

All 97 rooms are initialized in a flat array when the page loads:

```js
function initRooms() {
  const r = [];
  for (let fl = 1; fl <= 9; fl++)
    for (let pos = 1; pos <= 10; pos++)
      r.push({ id: fl * 100 + pos, floor: fl, position: pos, booked: false });
  for (let pos = 1; pos <= 7; pos++)
    r.push({ id: 1000 + pos, floor: 10, position: pos, booked: false });
  return r;
}
```

---

### Step 2: Travel Time Formula

Travel time between any two rooms is calculated as:

**Same floor:**
```
time = |position_A - position_B|
```

**Different floors:**
```
time = (position_A - 1) + |floor_A - floor_B| × 2 + (position_B - 1)
```

*Explanation:* Walk from room A to the lift `(position − 1)` steps, travel vertically `(2 min × floors apart)`, then walk from the lift to room B.

```js
function pairTime(r1, r2) {
  if (r1.floor === r2.floor) return Math.abs(r1.position - r2.position);
  return (r1.position - 1) + Math.abs(r1.floor - r2.floor) * 2 + (r2.position - 1);
}
```

**Total travel time** across a set of booked rooms is computed by sorting them by floor and position, then summing consecutive pairwise times:

```js
function totalTime(rooms) {
  const sorted = [...rooms].sort((a, b) =>
    a.floor !== b.floor ? a.floor - b.floor : a.position - b.position
  );
  let t = 0;
  for (let i = 1; i < sorted.length; i++) t += pairTime(sorted[i-1], sorted[i]);
  return t;
}
```

---

### Step 3: Single-Floor Best Pick (Sliding Window)

For a given floor, the algorithm finds the best `m` consecutive rooms using a **sliding window**:

```js
function pickFloor(avail, m) {
  const s = [...avail].sort((a, b) => a.position - b.position);
  let best = null, bestScore = Infinity;
  for (let i = 0; i <= s.length - m; i++) {
    const span  = s[i + m - 1].position - s[i].position;
    const score = span * 100 + s[i].position; // prefer tight cluster, then near lift
    if (score < bestScore) { bestScore = score; best = s.slice(i, i + m); }
  }
  return best;
}
```

This ensures rooms `[101, 102, 103]` are chosen over `[101, 105, 108]` — minimising spread and distance from the staircase.

---

### Step 4: Priority 1 — Same Floor

The algorithm first checks every floor with enough available rooms to fulfill the booking on its own:

```
For each floor with available rooms:
  If count >= n:
    Run sliding window → get best n rooms
    Track minimum travel time
```

If **any single floor** can satisfy the request, the floor with the lowest travel time is selected. **This always takes priority over cross-floor combinations.**

---

### Step 5: Priority 2 — Cross-Floor Optimization

If no single floor has enough rooms, the algorithm tries **all ways to distribute** the booking across multiple floors.

**5a. Generate partitions of n**

Split `n` into all ordered groups. For `n = 3`:
```
[3], [2,1], [1,2], [1,1,1]
```

```js
function partitions(n, max) {
  max = max ?? n;
  if (n === 0) return [[]];
  const res = [];
  for (let f = Math.min(n, max); f >= 1; f--)
    for (const r of partitions(n - f, f)) res.push([f, ...r]);
  return res;
}
```

**5b. Generate k-size floor combinations**

For each partition of size `k`, generate all `k`-size subsets of available floors:

```js
function combos(arr, k) {
  if (k === 0) return [[]];
  const [h, ...t] = arr;
  return [...combos(t, k-1).map(c => [h,...c]), ...combos(t, k)];
}
```

**5c. Generate permutations of room counts per floor**

For each floor subset, try all orderings of how many rooms to assign per floor (e.g. 2 to Floor 1 and 1 to Floor 3, or 1 to Floor 1 and 2 to Floor 3):

```js
function uniquePerms(arr) {
  if (arr.length <= 1) return [arr];
  const seen = new Set(); const res = [];
  for (let i = 0; i < arr.length; i++) {
    if (seen.has(arr[i])) continue; seen.add(arr[i]);
    const rest = arr.filter((_, j) => j !== i);
    for (const p of uniquePerms(rest)) res.push([arr[i], ...p]);
  }
  return res;
}
```

**5d. Evaluate all combinations and pick the best**

```js
for (const part of partitions(n)) {
  for (const floorCombo of combos(floors, part.length)) {
    for (const perm of uniquePerms(part)) {
      // pick perm[i] rooms from floorCombo[i] using sliding window
      // compute totalTime() of merged rooms
      // keep the combination with minimum time
    }
  }
}
```

---

### Step 6: Booking Confirmation & UI Update

Once the optimal rooms are found:
1. Their `booked` flag is set to `true` in the rooms array
2. They are highlighted **green** in the visual grid
3. A confirmation message shows the room IDs and total travel time in minutes
4. Previously booked rooms display as **orange** on subsequent renders

The grid re-renders on every booking, reset, or random action by rebuilding the DOM from the current rooms array using `innerHTML` and `createElement`.

---

## 🖥️ Features

| Feature | Description |
|---------|-------------|
| **Book Rooms** | Enter 1–5 and click Book; algorithm finds optimal rooms instantly |
| **Visual Grid** | All 97 rooms displayed floor-by-floor, left = closest to lift |
| **Room States** | White = Available · Green = Just Booked · Orange = Previously Booked |
| **Travel Time** | Every booking shows exact calculated travel time in minutes |
| **Random Occupancy** | Randomly occupies ~35% of rooms to simulate a real hotel state |
| **Reset** | Clears all bookings and returns every room to available |

---

## 📐 Example Walkthrough

**Scenario 1:** Available on Floor 1: `101, 102, 105, 106`. Guest requests 4 rooms.

1. Floor 1 has 4 available rooms ✅ — same-floor priority applies
2. Sliding window over `[101, 102, 105, 106]`:
   - Only 1 window of size 4 → `[101, 102, 105, 106]`, span = 5
3. Selected: `101, 102, 105, 106`
4. Travel time: `1 + 3 + 1` = **5 minutes**

**Scenario 2:** Only `101, 102` on Floor 1. Guest requests 4 rooms.

1. No single floor has 4 rooms → cross-floor mode
2. Try partition `[2, 2]`:
   - Floor 1 gives `101, 102` | Floor 2 gives `201, 202`
   - Travel: walk to lift `(1)` + 2 floors `(4)` + walk `(1)` + span on each floor `(1 + 1)` = **8 min**
3. All other 2+2 floor-pair splits are also evaluated; the one with the lowest total time wins

---

## 🏗️ Tech Stack

- **Language**: Vanilla JavaScript (ES6+)
- **UI**: Pure HTML5 + CSS3 — zero frameworks, zero dependencies
- **Algorithm**: Combinatorics in plain JS — partitions, combinations, permutations + sliding window
- **Deployment**: Single self-contained `.html` file — open in any browser

---

## 📁 File Structure

```
AdarshDixit_AssessmentSubmission/
├── README.md                  ← This file (solution explanation)
└── index.html                 ← Complete standalone app (open in any browser)
```

---

## 👤 Submitted By

**Adarsh Dixit**  
Assessment for: Software Development Engineer — Unstop  
Contact: dixitadarshdev@gmail.com
