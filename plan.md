# Mirror a GitLab group tree locally

Build a terminal application that mirrors a GitLab group's entire structure onto the local filesystem. Take the access 
token from `GITLAB_TOKEN` (prompt for it, hidden, if unset), the GitLab base URL from a `--url` parameter (falling back 
to `$GITLAB_URL`, then a prompt), and the numeric group id from a `--group` parameter. Recursively read every subgroup 
and repository under that group through the GitLab REST API (paginated via the `x-next-page` header) and reproduce the 
group's `full_path` hierarchy exactly under the current working directory, so even empty subgroups appear as directories. 
Clone every repository over SSH using ARO's embedded git.

Make it event-driven and parallel: discover repositories by storing each project (keyed by its `path_with_namespace`) 
into a repository, and let a repository observer perform the checkout, so discovery and cloning are decoupled through 
events; traverse subgroups and clone repositories with `parallel for each` for bounded concurrency. Define every object 
it passes around — the clone job, the GitLab group response, and each event payload — as an OpenAPI component schema and 
pull them in one validated step with typed extraction (`Extract the <x: Schema> from ...`) instead of field-by-field. 
Wire the whole flow through custom events and handlers (group discovered, repository found, repository cloned) plus the 
repository observer.

Render progress reactively with templates as a dynamically sized square grid: show a hollow square `◻︎` for each 
repository the moment it is found and added to the repository, and flip it to a filled square `◼︎` when its clone 
completes, re-rendering the grid in place on every state change.
