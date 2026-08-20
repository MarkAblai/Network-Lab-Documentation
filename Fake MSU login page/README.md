# COB Lab Decoy Portal

A single-file decoy login page for a networking classroom exercise. Students run a port scan against the lab network, find an open web server, browse to it, and get a harmless "gotcha" reveal instead of a real login.

Built as a lab assistant teaching aid for an ITC networking course at Missouri State University.

## Not affiliated with Missouri State University

**This is not an official Missouri State University site, service, or login page.** It is a classroom teaching prop. It is not operated, endorsed, or approved by Missouri State University, the College of Business, or the university IT department. It has no connection to BearPass or any real authentication system.

The Missouri State and College of Business marks in this page are university brand assets used for an internal classroom demonstration only. They remain the property of Missouri State University. If you fork or reuse this project, replace the branding with your own.

## It does not capture credentials

This matters more than anything else in the repo, so it is worth being explicit:

- No backend, no server-side code, no database
- No `action` attribute on the form and no `fetch`, `XMLHttpRequest`, or `navigator.sendBeacon` anywhere
- No `localStorage`, `sessionStorage`, or cookies
- No analytics, no external requests of any kind

Whatever gets typed into the form is discarded when the page state changes. The reveal screen says so directly, so students are told immediately that nothing was taken.

If you modify this page, do not add credential logging. Collecting classmates' real passwords is a serious problem regardless of intent, and it turns a teaching exercise into an incident.

## What it does

**Login state.** A plausible institutional portal. University branding, a maintenance notice, a username and password field, and a footer advertising a fake host and server version that matches what a scanner banner would show.

**Reveal state.** Any submit, Enter keypress, or click on the help links wipes the page to a dark terminal. The message is delivered as scanner-style output with a port table pointing at the row the visitor came in through, then explains the lesson: an open port is an invitation to look, not a reason to trust.

A back button returns to the login so the next student gets a clean page.

## Files
