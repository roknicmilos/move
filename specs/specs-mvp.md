# Project Specifications

This project should be a PWA that tracks my workouts. I should be able to install it on my Android like a
native app. It should have a backend REST API with a database and frontend that talks to that API.

Information in this document is just an idea, a draft. This means that it should be reviewed and better alternatives
should be considered and suggested. Terminology and component (classes, variables, functions, etc.) names are also
things that should be considered for improvements.

## Workouts

The project should support 2 different workouts represented by a single Workout entity. These 2 workouts are:

- Flow: Main Low Back Mobility Session
- Mobility Flow: Supporting Low Back Mobility Session

### Entities

- Workout
    - name: str
    - short_name: str
    - instructions: str
    - purpose: str (optional)
    - exercises: Exercise collection
- Exercise
    - workout: Workout (FK)
    - name: str
    - description: str
    - sets: int (optional)
    - reps: int (optional)
    - duration_sec: int (seconds) optional
    - load_kg: float (kilograms) optional
- WorkoutSession
    - workout: Workout
    - started_at: datetime
    - ended_at: datetime
    - exercises: ExerciseSet collection
- ExerciseSet
    - exercise: Exercise
    - load_kg: float (kilograms) optional
    - duration_sec: int (seconds) optional

### Fixtures (Seeds)

See [workouts.yaml](../seeds/workouts.yaml) and [exercises.yaml](../seeds/exercises.yaml) for the full list of
Workout and Exercise seed data (referenced by `workout` FK on each Exercise).

## Web App Features

- User should be able to have an overview of all workouts. Selecting a workout leads to that workout page with details
  about that workout and list of all exercises. Selecting an exercise leads to that exercise page with details about
  that exercise.
- User should be able to record a workout. When user selects a workout, there should be a button to start a workout. It
  opens a page that clearly shows that workout is in progress. Each exercise has a checkmark that user can select after
  completing that exercise. Starting a workout automatically sets started_at for that workout session. There should be a
  button to end the workout that sets ended_at for that workout.
- User should be able to see a history of their workouts. They should be able to select a view for this history: weekly
  and monthly. Default is weekly.

## Authentication

The app has predefined users. Each user has a username and password. There is no registration, only login. Authenticated
user session should expire after 2 weeks.

## Architecture

### Backend

TODO

### Frontend

TODO
