# Project Specifications

This project should be a PWA that tracks my physical activity. I should be able to install it on my Android like a
native app. It should have a backend REST API with a database and frontend that talks to that API.

Information in this document is just an idea, a draft. This means that it should be reviewed and better alternatives
should be considered and suggested. Terminology and component (classes, variables, functions, etc.) names are also
things that should be considered for improvements.

## Workouts & Activities

The project should support multiple types of workouts / activities. Activities usually imply walking, hiking, swimming,
running/jogging and similar. These activities usually do not have any parts like workouts that have specific exercises.

### Activities

- Walking
    - started_at: datetime
    - ended_at: datetime
    - steps: int
- Running
    - pace: time (maybe int for seconds instead time?)
    - started_at: datetime
    - ended_at: datetime
    - distance_km: float
- Hiking
    - pace: time (maybe int for seconds instead time?)
    - started_at: datetime
    - ended_at: datetime
    - distance_km: float
    - lowest_elevation_m: int (meters)
    - highest_elevation_m: int (meters)

Make sure to crate a shared Activity class that holds shared fields and methods.

These activities are usually logged by Garmin (and Strava app). It would be great if this project can sync that data
locally and create the above entities (objects) automatically.

### Workouts

- Workout
    - name: str
    - short_name: str
    - instructions: str
    - purpose: str (optional)
    - exercises: Exercise collection
- Exercise
    - name: str
    - description: str
- WorkoutSession
    - workout: Workout
    - date: date
    - exercises: ExerciseSet collection
- ExerciseSet
    - exercise: Exercise
    - load_kg: float (kilograms) optional
    - duration_sec: int (seconds) optional

#### Fixtures (seeds)

- Workouts:
    1. name: Low Back Ability Main Workout
       short_name: LBA Main
       description: The main workout session for lower back strengthening.
    2. name: Low Back Ability Supporting Workout
       short_name: LBA Supporting
       description: The supporting workout session for lower back strengthening, plus calisthenics exercises.
- Exercises:
    1. name: Sled Pull
       instructions: Start with just BACKWARDS sled for the first few sessions. Gradually progress to include Sled
       PUSH/PULL COMBO up to 2x/week. Sled PUSH involves gentle spinal compression that we want to slowly build
       tolerance to. If a sled isn’t available, walk backward on a treadmill (turned off) or simply practice walking
       backward. Pay attention to your footing to avoid injury.
       purpose: The Sled is a complete lower body exercise that is very low back friendly. The Sled PUSH is a great
       middle step back to traditional lifts like back squat. Training the legs with tolerable spinal compression. The
       sled PUSH actually has been a great relief tool for many people during a flare up (take it slow).
    2. name: Tibialis Raises
       instructions: Stand upright, or with a slight bend at the hips for more ease. Begin with the hips against a wall
       and the heels standing about 12 inches away from the wall. Lift your toes up and towards your shins, engaging the
       muscles of the anterior lower leg.
       purpose: Our “Ground-Up” rebalancing begins at the first common weak link: The FRONT of the ankle. Weakness can
       often be seen after prolonged nerve pain, sometimes even presenting as foot drop. Building strength in the
       ankles & lower legs is our first step in finding pain-free evidence as we work our way up. Weak ankles leave the
       knees vulnerable which can affect our true low back training.

## Architecture

### Backend

TODO

### Frontend

TODO

## Third party APIs

- **Google Fit** service records daily steps. If it's possible, this project should fetch data from Google Fit and
  create/update Walking records automatically.
- **Garmin** service records multiple activities. If it's possible, this project should fetch data from Garmin and
  create/update different activity records automatically.
- **Strava** service records multiple activities. If it's possible, this project should fetch data from Strava and
  create/update different activity records automatically.
- Walking steps should not overlap existing ones, regardless from which service (third party) data was fetched. For
  example, if there are Walking records in DB that overlap with newly fetched walking records from Garmin, the project
  should recognize that and omit the parts that overlap. If there is existing Walking from 10:00 AM until 11:00 AM, and
  newly fetched walking data is from 10:30 AM until 11:30 AM, the existing Walking should be updated to be from 10:00 AM
  until 11:30 AM.
