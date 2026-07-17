# Work Session "Current Active Session" Endpoint Request

## Problem Context
When the Boblte Field Force mobile app restarts, reloads, or experiences a local state clear, it currently loses track of the field executive's open work session because there is no API endpoint to verify if they are actively clocked in on the backend upon boot. 

Because the app defaults its state to "Not Started", the user will mistakenly see the "Hold to Clock In" button. When the user attempts to clock in, the backend correctly rejects the request with a `409 Conflict: A work session is already open for this field executive`.

The user is then effectively stuck — they cannot clock in, but they also have no UI element to clock out because the app does not know they have an active session.

## Temporary Workaround (Currently in Mobile App)
To unblock users, the mobile app now catches the `409 Conflict` (specifically looking for the "already open" error message string) when they attempt to clock in, and artificially reconstructs a local "Active" session so that the "Clock Out" button is revealed. The app can successfully clock out because the `POST /work-sessions/executives/:fieldExecutiveId/clock-out` endpoint does not strictly require the `sessionId` from the client.

## Requested Backend Implementation

To fix this robustly, we need a simple GET endpoint to fetch the currently active session for a specific field executive.

**Endpoint details:**
`GET /work-sessions/executives/:fieldExecutiveId/current`
*(or equivalent within the existing `WorkSessionsController`)*

**Behavior:**
- Return the active/open `WorkSession` object if one exists (i.e. `status === 'ACTIVE'` or `clockOutAt === null`).
- Return a `404 Not Found` (or empty `200 OK` with `null`) if there is no active session.

**Consumer:**
The Field Force App's `WorkSessionNotifier` will call this endpoint upon initialization (`build()` method) to seamlessly restore the user's active session state across app restarts.
