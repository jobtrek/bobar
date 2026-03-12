# "Bobar" initial specification

Bobar will be a simple application to allow user to make feedback on staging deployments (like vercel bar). The main idea is to provide a simple script that dev's can
add to their source code. The script will display a simple floating "comment" button overlay on the deployed app. The final user testing the app can trigger this button
and write comments or click on the app to indicate an html element. The script then record a screenshot, the mouse position, screen size, html elements clicked... Then, the
script upload all the data on the pull request corresponding to deployment actually tested.

A second feature of the app should be a user friendly overview of the roadmap and changelog of an application deployed on github. There is to main features.
First, querying the github relases and displaying them on a user friendly timeline. listing changes, and all respolved issues by the release.
Another feature would be to display the completion of the actually running milestone on a user friendly way, with completion...
Finally, a page should display all issues tagged with a "vote" label, the idea is to allow users to vote for features they want. The devs create an issue on github that describe the issue. He label the issue.
Then the app query all labeled issue and display them on a votation page. User vote. Then the app update the issue indicating the number of upvotes/downvotes.

The app needs a simple backend with authentication to configure the system (repo name, observed labels, deployemnt branches schema to determine on wich pr comment from toolbar...).

The toolbar should detect the actual deployemnt type (from url). If this is a pull request deployment, put a comment of the pr with the informations from the user. If it is a "generic" deployment, like dev, staging or production, the app shoul open a issue on the repo with the user comment and infos, tagging the issue with label to know the env that triggered that.

On the toolbar, when the user trigger a comment, he should have a form to indicate informations, also some predefined checkboxes/selects to allow dev to query informations. The dev should be able to configure all the feeddback form from the app backent...
