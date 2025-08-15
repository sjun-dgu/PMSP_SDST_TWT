README

Overview
--------
This repository defines a small scheduling environment for unrelated parallel machines with sequence-dependent setup times. It contains three core Python classes:

- Job: a production order with due date, weight, etc.
- Machine: a processing resource that holds assigned jobs, setup/processing times, and availability.
- Instance: a container that bundles jobs/machines with their processing and setup time matrices.

Because Python objects aren’t JSON-serializable by default, the code also shows how to convert these classes to plain dictionaries and then to JSON.

Features
--------
- Clear object model for jobs, machines, and problem instances
- Utilities to compute expected completion times
- Safe to_dict() methods for JSON export (no circular references)
- Example script to write an Instance to instance.json

Data Model
----------
Job
  - Core fields: ID, due, weight, family
  - Dynamic fields: complete, start, end, assignedMch, priority
  - Helpers:
    - get_setups(mch_list) → min/max/avg setup time across machines
    - get_ptimes(mch_list) → min/max/avg processing time across machines
    - get_min_comp(mch_list) → earliest possible completion time across machines

Machine
  - Core fields: ID, available, priority
  - Schedule state: assigned (list of Job), schedules (list of bars/segments), setup, ptime
  - Helpers:
    - get_setup(job)
    - get_ptime(job)
    - process(job) (advances availability; appends to schedule)
    - get_min_comp(job_list)

Instance
  - Core fields: numJob, numMch, job_list, machine_list
  - Matrices:
    - ptime[machine_id][job_id]
    - setup[machine_id][prev_job_id][next_job_id]
  - Flags: with_setup, family_setup, identical_mch
  - Helpers: findJob, findMch, deepcopy, make_subprob

JSON Serialization
------------------
Python’s json module cannot serialize custom classes directly. Each class therefore implements to_dict() which converts the object into a JSON-friendly dictionary using only base types (int, float, str, bool, list, dict, None).

Design choices:
- No object references in JSON: store references by ID (e.g., assigned is a list of job IDs; assignedMch becomes a machine ID or -1).
- Matrices are kept as nested lists (ptime, setup).
- Complex fields (e.g., Bar in schedules) should be converted to strings or their own dicts.

Example: Instance → JSON

    import json
    json_str = json.dumps(inst.to_dict(), indent=2)
    with open("instance.json", "w") as f:
        f.write(json_str)

Example JSON Snippet

    {
      "numJob": 3,
      "numMch": 2,
      "jobs": [
        { "ID": 0, "due": -1, "weight": 0, "family": null, "complete": false,
          "start": -1, "end": -1, "assignedMch": -1, "priority": 0 }
      ],
      "machines": [
        { "ID": 0, "available": 0, "assigned": [], "setup": null,
          "ptime": null, "schedules": [], "priority": 0 }
      ],
      "ptime": [[1,2,3],[2,1,4]],
      "setup": [[[0,1,2],[1,0,3],[2,3,0]], [[0,2,1],[2,0,4],[1,4,0]]],
      "with_setup": true,
      "family_setup": false,
      "objective": "None",
      "identical_mch": false
    }

Deserialization (Optional)
---------------------------
To rebuild objects from JSON, implement complementary from_dict() constructors. Use IDs to map references back to objects and restore ptime/setup matrices.

Quick Start
-----------
1. Install (Python 3.9+ recommended)
    pip install -r requirements.txt  # if provided; stdlib is enough for core usage

2. Run example
    python main.py
   This will build a small Instance and write instance.json.

3. Load from JSON (if you added from_dict)
    import json
    from model import Instance
    with open("instance.json") as f:
        data = json.load(f)
    inst = Instance.from_dict(data)

Design Notes & Conventions
--------------------------
- ID-based references: JSON doesn’t store object pointers; IDs keep things stable and portable.
- No circular JSON: avoid serializing nested objects that point back to parents.
- Matrices are machine-major:
  - ptime[m][j] → processing time of job j on machine m
  - setup[m][i][j] → setup time on machine m from job i to job j
- Extensibility:
  - If you use a Bar schedule element, define Bar.to_dict()/from_dict().
  - If you use a callable or function handle for objective, serialize by name and restore by import/lookup.

Common Pitfalls
---------------
- Serializing live object references will fail — convert to IDs.
- Mismatched matrix dimensions: Ensure ptime has shape [numMch][numJob] and setup has [numMch][numJob][numJob].
- Restoring order: When rebuilding, use IDs (not array position) to map back jobs and machines.

Testing Tips
------------
- Create a tiny instance (e.g., 2 machines × 3 jobs).
- process() a couple of jobs to populate assigned, available, and start/end.
- Export → import → verify:
  - Same number of jobs/machines
  - Same IDs and attributes
  - assigned lists restored
  - ptime/setup match originals

License
-------
MIT (or your preferred license).

Changelog
---------
v0.1.0: Initial classes and JSON export via to_dict(); optional from_dict() guide.

FAQ
---
Q. Why not use pickle?
A. Pickle is Python-specific and less portable/transparent. JSON is language-agnostic and easier to inspect.

Q. Can I add new fields?
A. Yes—just include them in to_dict() and handle them in from_dict().

Q. How do I serialize numpy arrays?
A. Convert to lists before storing in JSON (arr.tolist()), and rebuild with np.array() on load if needed.
