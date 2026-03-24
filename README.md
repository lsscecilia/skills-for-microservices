# skills for microservices

Collections of skills to allow AI agents to understand and work with microservice better


## `/be-api-trace` — Backend API Tracer                                                                                                           
                                                                                                                                                    
  For when you need to figure out what backend changes are needed based on a frontend API call.                                                     
                                                                                                                                                    
  **Trigger**: Paste a browser DevTools network call (method, URL, request, response)                                                               
                  
  **What it does**:                                                                                                                                 
  1. Parses the API contract from the network call
  2. Searches the current repo for the route handler — tells you if it's in a different service                                                     
  3. Maps what the FE is rendering from the response, flags mismatches against any TypeScript types you paste                                       
  4. Maps response fields to DB tables/columns                                                                                                      
  5. Outputs a scoped backend change plan (DB migration + service changes + before/after contract diff)                                             
  6. Drafts the one-liner to send the FE engineer describing what changed                                                                           
                                                                                                                                                    
  **You need**: DevTools network call. Optionally: FE TypeScript types, your DB schema file.                                                        
                                                                                                                                                    
  ---                                                                                                                                               
                  
  ## `/ms-spec` — Microservice Spec Generator

  For documenting your full microservice architecture so AI agents (and new engineers) have complete context.                                       
   
  **What it does**:                                                                                                                                 
  1. Discovers all services from the repo, docker-compose, or gateway config
  2. Builds a system map — who calls whom, what events flow where, who owns what data                                                               
  3. Scans for **domain pollution** (biz logic that doesn't belong in a service) and **legacy/dead code** (deprecated endpoints, dual               
  implementations, feature flags, orphaned DB columns)                                                                                              
  4. Grills you with targeted questions — one category at a time — until every gap is filled or explicitly marked `[UNKNOWN]`                       
  5. Writes `docs/architecture/ARCHITECTURE.md` + one `<service-name>.md` per service                                                               
                                                                                                                                                    
  **Each spec file includes**: purpose, business domain ownership (with ⚠ for misplaced logic), API endpoints, DB schema, events, outbound calls,   
  key business rules, tech debt, legacy code, and a "Context for AI Agents" section.                                                                
                                                                                                                                                    
  **How to use the output**: Reference the `.md` files at the start of any Claude session — one file gives full context for that service without
  re-explaining anything.

  ---

  ## `/update-all` — Update All Repos

  For when you want to sync all microservice repos at once — checkout master/main and pull latest.

  **What it does**:
  1. Asks for (or accepts) the parent folder containing all your repos
  2. Dynamically discovers every git repo inside it
  3. Detects whether each uses `master` or `main`
  4. Checks out the default branch and pulls latest for each
  5. Reports what succeeded and flags anything that needs manual attention

  **You need**: The path to your parent folder (e.g. `~/projects`).

