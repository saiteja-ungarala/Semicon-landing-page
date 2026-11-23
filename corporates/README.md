# Corporates (Teams) landing page - v6 (2026-11-20)

Static page: `index.html` + `assets/` + `robots.txt`. No build step.

Deploy: copy the contents of this folder to `public_html/corporates/` on semiconlabs.com
(so the page is https://semiconlabs.com/corporates/). The form posts to the vigyan API
(`POST /v1/leads`); CORS already allows semiconlabs.com.

Form fields: Name, Email, Contact Number (10 digits), Company Name, No. of Seats Required (min 2),
Location, Website, Few lines (the last three are optional).

After uploading, submit ONE real test enquiry and confirm it appears in the leads list.
