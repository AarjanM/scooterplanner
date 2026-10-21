# ScooterPlanner

Route planning and navigation for Dutch mopeds, which Dutch traffic law sends onto different roads and paths than bicycles. This glossary fixes the legal vocabulary the routing rules are built on.

## Language

### Vehicles

**Bromfiets**:
A moped with a yellow licence plate, limited to 45 km/h. Rides on the carriageway unless a bromfietspad is signed.
_Avoid_: Scooter, brommer, moped (ambiguous: also used for snorfiets)

**Snorfiets**:
A light moped with a blue licence plate, limited to 25 km/h. Rides on cycle paths unless a local rule sends it to the carriageway.
_Avoid_: Scooter, light moped, mofa

**Speed pedelec**:
An electric bicycle assisting up to 45 km/h. Dutch law classes it as a Bromfiets: yellow plate, Rijbaan or Bromfietspad, never a Verplicht fietspad.
_Avoid_: Fast e-bike, S-pedelec

**Fiets**:
A bicycle or a pedelec assisting up to 25 km/h, including fatbikes. Rides on any cycle path and is the only class Google Maps already routes correctly.
_Avoid_: Bike, e-bike (ambiguous with Speed pedelec)

**Vehicle class**:
The legal category a rider's vehicle falls in (Fiets, Snorfiets or Bromfiets), which selects the set of roads and paths a route may use. A rider picks a vehicle type such as Speed pedelec; the app maps it to its Vehicle class.
_Avoid_: Mode, profile (reserved for the routing engine's internal term)

### Infrastructure

**Rijbaan**:
The carriageway shared with motor traffic. The default surface for a Bromfiets.
_Avoid_: Road, street

**Bromfietspad**:
A cycle path signed G12a (fiets/bromfietspad) that cyclists, snorfietsen and bromfietsen may all use.
_Avoid_: Moped lane, shared path

**Verplicht fietspad**:
A mandatory cycle path signed G11 (blue round sign). Open to cyclists and snorfietsen, closed to bromfietsen.
_Avoid_: Bike lane, cycleway (too generic: hides the G11 vs G12a distinction that matters here)

**Onverplicht fietspad**:
An optional cycle path signed G13. Open to cyclists only; a Snorfiets may use it only with the engine off.

**Autoweg**:
A road signed G3 where mopeds are banned regardless of speed limit. Together with autosnelweg (G1) these are always excluded.
_Avoid_: Highway, trunk road

**Rijbaan zone**:
A municipal area where snorfietsen must leave the cycle path and ride on the Rijbaan, such as inside Amsterdam's A10 ring.
_Avoid_: Snorfiets ban, helmet zone

### Routing

**Legal route**:
A route that uses only infrastructure the chosen Vehicle class may ride on under Dutch traffic law.
_Avoid_: Safe route, allowed route

**Route plan**:
A Legal route computed from an origin to a destination before riding, shown on a map.
_Avoid_: Trip, journey, directions

**Turn-by-turn navigation**:
Live guidance along a Route plan while riding, with the rider's position tracked and instructions spoken.
_Avoid_: Navigation (alone: ambiguous with Route plan)
