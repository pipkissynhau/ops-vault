# 02-Cloud Migration Methodology: Full, Incremental, Cutover, Verification, Rollback

## Document Notes
Organize common methodologies in cloud migration projects, focusing on key phases such as full synchronization, incremental synchronization, final cutover, data verification, and rollback design. Suitable for knowledge base documentation, interview review, and migration project planning.

## Tags
#CloudMigration #MigrationMethodology #FullSync #IncrementalSync #Cut #DataValidation #Retract #Transport

---

## I. Why Cloud Migration Needs a Methodology

Cloud migration is not simply data replication, nor is it just moving virtual machines or databases "over there".  
Any production migration will inevitably involve the following goals:

1. **Minimize business downtime**
2. **Ensure data consistency**
3. **Reduce business risks from migration failures**
4. **Make the migration process verifiable, reversible, and deliverable**

Therefore, a mature migration project typically doesn't perform a single-time copy, but instead executes in multiple phases according to methodology. The most core keywords are:

- Full
- Incremental
- Cutover
- Verification
- Rollback

These terms apply almost universally to all migration types, including:

- Host migration
- Database migration
- Object storage migration
- Big data platform migration
- Application system migration

---

## II. Standard Migration Project Timeline

A standard migration project can typically be abstracted into the following timeline:

1. **Migration Assessment**
2. **Target Environment Preparation**
3. **Full Synchronization**
4. **Incremental Synchronization**
5. **Cutover Preparation**
6. **Final Cutover**
7. **Post-Migration Verification**
8. **Observation and Rollback Assurance**

If further summarized, it can be compressed into one sentence:

> **First move historical data, then synchronize changing data, and finally complete the switch in a controlled window, while ensuring verification and rollback.**

---

## III. Full Synchronization

### 1. What is Full Synchronization

Full synchronization refers to the first-time complete replication of all existing base data of the migration object to the target end.

The objects of full synchronization vary by scenario:

#### Host Migration
Full synchronization typically includes:
- System disk
- Data disk
- Operating system
- Applications
- Configuration files

#### Database Migration
Full synchronization typically includes:
- Table structure
- Indexes
- Historical business data
- Initial base data

#### Object Storage Migration
Full synchronization typically includes:
- Historical files
- Images
- Videos
- Attachments
- Archive objects

### 2. Characteristics of Full Synchronization

#### 1) Largest Data Volume
The full synchronization phase is usually the stage with the largest data volume and longest duration in the entire migration process.

#### 2) Suitable for Early Execution
Because full synchronization is time-consuming, it is generally not executed on the day of the formal switch, but rather in advance.

#### 3) Completion of Full Synchronization Does Not Mean Migration Completion
This is a critical point.  
If the source system is still online, new data or changed data will continue to be generated after full synchronization completes, which won't automatically appear on the target end, so incremental synchronization is still needed afterward.

### 3. What to Focus on During Full Synchronization

#### Data Scale
- Total capacity
- Number of files
- Number of tables, records
- Presence of massive small files or super large objects

#### Network Bandwidth
- Current link bandwidth adequacy
- Whether using public internet, dedicated line, or VPN
- Whether traffic limiting is needed

#### Target End Capacity
- Disk space sufficiency
- Database space adequacy
- Object storage Bucket readiness

#### Time Window
- Full synchronization duration
- Whether long transmission is allowed
- Whether batch execution is needed

### 4. Typical Risks of Full Synchronization

- Mismatched source and target end capacities
- Network instability causing full synchronization interruption
- Long full synchronization duration affecting subsequent migration plans
- Low efficiency due to massive small files
- Structural migration completed but data migration failure

### 5. One-Sentence Understanding of Full Synchronization
> **Full synchronization is responsible for moving "historical stock" first.**

---

## IV. Incremental Synchronization

### 1. What is Incremental Synchronization

Incremental synchronization refers to synchronizing new, modified, or deleted changes from the source end to the target end after full synchronization completes.

This is a critical phase in migration projects, as data will continue to change as long as the source system is operational.

### 2. Why Incremental Synchronization is Mandatory

Because there is typically a time gap between full synchronization and the formal switch.  
During this period, the source system continues to generate changes, such as:

#### Database Scenario
- New orders added
- User information modified
- Historical data deleted

#### Host Scenario
- New logs continue to be written
- Configuration continues to change
- New files continuously generated

#### Object Storage Scenario
- New images uploaded
- Attachments updated
- Objects deleted or overwritten

If only full synchronization is done without incremental synchronization, the target end will definitely fall behind the source end at the time of the formal switch.

### 3. Core Value of Incremental Synchronization

#### 1) Compress Downtime Window
Without incremental synchronization, the downtime window would be long as the full synchronization after the switch would need to be re-synced.  
With continuous incremental synchronization, the final cutover only needs to synchronize the last small segment of differential data, significantly shortening the downtime window.

#### 2) Reduce Cutover Risks
Through continuous incremental synchronization, issues can be exposed in advance, such as:
- Logs falling behind
- Incremental link anomalies
- Some objects synchronization failure
- Data conflicts

#### 3) Support Multi-Round Drills
Incremental synchronization can last for a period, transforming the migration project from a one-time move to an "observable, verifiable, and gradually convergent" process.

### 4. Common Implementation Approaches for Incremental Synchronization

Different migration objects have different approaches for incremental synchronization.

#### Database Migration
Common dependencies:
- Binlog
- Redo/archive logs
- CDC
- Transaction log parsing

#### Host Migration
Common dependencies:
- File change detection
- Disk block change synchronization
- Incremental replication mechanism
- Agent continuous collection

#### Object Storage Migration
Common dependencies:
- New object scanning
- Modification time comparison
- Directory/object list difference comparison
- Incremental task rerun

### 5. What to Focus on During Incremental Synchronization

#### Synchronization Latency
- Current latency level
- Whether it can catch up
- Whether there is continuous backlog

#### Data Order
- Whether incremental data is applied in order
- Whether there is out-of-order risk
- Whether events might be lost

#### Delete and Update Semantics
- Whether deletions are synchronized
- Whether overwrite writes retain data
- Whether object metadata changes are synchronized

#### Business Write Pressure
- Whether incremental can catch up during peak hours
- Whether it needs to avoid business peaks

### 6. One-Sentence Understanding of Incremental Synchronization
> **Incremental synchronization is responsible for continuously catching up with "changes after full synchronization."**

---

## V. Cutover

### 1. What is Cutover

Cutover refers to the process of formally switching business from the source end to the target end.  
It is not the action of "starting migration," but rather the action of "completing migration and switching production entry points."

Many migration projects' true risks are not in the full synchronization phase, but in the cutover phase.

### 2. Why Cutover is Critical

Because cutover means:

- Switching business entry points
- Switching write links
- Switching user traffic
- Switching production responsibility

Any small issue during this phase can directly impact business operations.

### 3. What Cutover Typically Involves

#### Database Migration Scenario
- Stop source database writes or pause business write requests
- Execute final incremental synchronization
- Switch connection strings to the target database
- Resume business writes on the target database

#### Host Migration Scenario
- Stop source end application services
- Execute final synchronization
- Start target host and target application
- Switch DNS, VIP, SLB, EIP, or routing

#### Object Storage Migration Scenario
- Stop source end writes
- Complete the final round of object incremental synchronization
- Switch domain names, CDN backsource or download addresses
- Verify if all application references have been redirected to the new storage

### 4. What is a Cutover Window

A cutover window is the predetermined time period for officially switching business operations.  
This window generally needs to be confirmed in advance with the business party, development team, testing team, and network team.

A reasonable cutover window should clearly answer:

- When to stop writes
- When to start the final synchronization
- When to start the entry switch
- When to start business verification
- When to announce the cutover completion
- If failed, when to start rollback

### 5. What to Focus on During Cutover

#### Write Stop Control
Must clearly define:
- Who will stop writes
- How to stop writes
- Whether it's application stop, interface circuit breaker, or database read-only

#### Switching Order
The switching order must be clear, for example:
- Stop business first
- Then perform final synchronization
- Then start the target end
- Then switch traffic
- Then verify

#### Business Dependencies
Need to confirm:
- Whether there are hidden dependencies not switched
- Whether there are old configurations pointing to the source end
- Whether there are unadjusted scheduled tasks, message queues, or caches

#### Change Communication
During the cutover window, there must be a clear owner, executor, observer, and rollback decision-maker.

### 6. One-Sentence Understanding of Cutover
> **Cutover is responsible for switching the "production entry" from the source end to the target end.**

---

## Six. Verification

### 1. What is Verification

Verification refers to checking the data, structure, functionality, and access results of the source and target ends before and after migration or switching, to confirm whether the migration was correctly completed.

Verification is not an action, but a complete set of verification processes.

### 2. Why Must Verification Be Done

Because "migration task shows success" does not equal "business is really fine".

For example:
- File copy succeeded, but permissions are wrong
- Table data has been synchronized, but indexes are incomplete
- Application can start, but connection is still to the old address
- Object count is consistent, but metadata is lost
- Database can connect, but business SQL execution is abnormal

### 3. What Categories Does Verification Usually Fall Into

#### 1) Data Verification
Verify whether the data of the source and target ends are consistent.

Common dimensions include:
- Record count
- File count
- Total data volume
- Hash value
- Sampling comparison of key fields

#### 2) Structural Verification
Verify whether the structural layer is consistent, for example:
- Table structure
- Indexes
- Partitions
- Storage type
- Directory structure
- Permission configuration

#### 3) Functional Verification
Verify whether business functions are normal, for example:
- Login is normal
- Order placement is normal
- Upload/download is normal
- Query is normal
- API is normal

#### 4) Performance Verification
Verify whether the target end has sufficient performance after migration, for example:
- Response latency
- Throughput capacity
- CPU, memory, disk, water level
- Database slow query situation

### 4. When Should Verification Be Done

#### Before Migration
- Record baseline
- Statistic source data volume and key metrics
- Determine subsequent comparison basis

#### After Full Migration
- Verify whether basic data has been copied
- Verify whether the target end can start and be accessed

#### During Incremental Migration
- Continuously monitor synchronization delay and errors
- Do sampling comparison for key business data

#### After Cutover
- Do final data consistency verification
- Do business function verification
- Do entry switch verification

### 5. One-Sentence Understanding of Verification
> **Verification proves that "migration is done" is not just surface success, but truly usable, reconcilable, and deployable.**

---

## Seven. Rollback

### 1. What is Rollback

Rollback is the process of switching business back to the source end when migration switching fails or serious issues are discovered after migration.

Migration plans without rollback design usually carry very high risks.

### 2. Why Must Rollback Be Designed

Because migration is not an experiment, but a production change.  
As long as it's a production change, failure scenarios must be considered.

Typical failure scenarios include:
- Target end business startup failure
- Configuration missing
- Data inconsistency
- Application errors
- Performance drops significantly
- External dependencies not aligned
- Key function verification failure

### 3. What Problem Does Rollback Essentially Solve

Rollback solves:

> **If the new environment can't be switched, can business recover quickly?**

It's essentially the fallback mechanism in the migration plan.

### 4. What to Prepare Before Rollback

#### Define Rollback Conditions
Must define in advance:
- What situations require rollback
- Who has the authority to decide rollback
- What are the rollback trigger thresholds

For example:
- Core functions unavailable for over 10 minutes
- Key data verification failure
- Interface error rate continuously rising
- Target end performance can't meet basic business needs

#### Maintain Source End Availability
The source end can't be completely destroyed or irrecoverable before switching.  
Usually need:
- Preserve original instance
- Preserve original database
- Preserve original access entry configuration
- Preserve source end's latest recoverable state

#### Define Rollback Steps
Rollback can't be decided on the spot, must be clearly written in advance, for example:
- Stop target end writes
- Switch entry back to source end
- Restore source end service
- Verify source end status
- Notify business recovery

### 5. What is the Biggest Challenge of Rollback

#### Target End Already Has New Data
This is the most troublesome situation.  
If there's new writing after switching to the target end, rollback needs to consider:
- Whether to discard new data on the target end
- Whether to synchronize new data back to the source end
- Whether to allow short-term data rollback

Therefore, many migration plans have a critical observation window after switching, where write scale, permissions, or switching pace are strictly controlled during the observation window.

### 6. One-Sentence Understanding of Rollback
> **Rollback ensures business can be safely switched back to the source end when migration fails.**

---

## Eight. Why "Full, Incremental, Cutover, Verification, Rollback" Must Be Viewed Together

Many people memorize these terms separately, but when actually doing migration, they form a complete loop:

- **Full**: First move historical data
- **Incremental**: Continuously fill changes
- **Cutover**: Officially switch business entry
- **Verification**: Confirm migration results are correct
- **Rollback**: Provide a fallback for failure

Missing any link makes the migration plan fragile.

For example:

#### Only Full, No Incremental
Target end is always behind the source end

#### Has Incremental, No Cutover Plan
Data is there, but business can't switch

#### Has Cutover, No Verification
Problems are discovered after switching

#### Has Switching, No Rollback
Once failed, can only hard-press

### One-Sentence Understanding of Overall Relationship
> **These five links together form a migration loop that is executable, verifiable, and reversible.**

---

## Nine. How These Five Links Are Reflected in Different Migration Types

### 1. Host Migration
- Full: Copy system disk, data disk, application environment
- Incremental: Synchronize subsequent changes
- Cutover: Stop application, final synchronization, start target host
- Verification: Service normality, port normality, data normality
- Rollback: Switch back to original host and original entry

### 2. Database Migration
- Full: Copy historical table structure and data
- Incremental: Synchronize binlog/log changes
- Cutover: Stop writes, final synchronization, switch connection string
- Verification: Table records, business functions, key data consistency
- Rollback: Switch back to original database connection

### 3. Object Storage Migration
- Full: Copy historical objects
- Incremental: Synchronize new or modified objects
- Cutover: Switch domain name/CDN/access address
- Verification: Object count, size, hash, access path
- Rollback: Switch back to original object storage entry

---

## Ten. Must Ask Questions When Designing a Migration Plan

### 1. About Full Migration
- What is the total data volume
- How long is the estimated full migration time
- Can it be done in advance
- Do we need to split into batches

### 2. About Incremental
- What is the incremental mechanism
- Is delay controllable
- Can it catch up during peak hours
- Does it support deletion, overwrite, metadata synchronization /think

### 3. About Cut-over
- Who executes the stop-write
- What is the switch order
- What are the impact ranges
- How long is the cut-over window

### 4. About Validation
- What data to validate
- What is the validation scope
- Who conducts the acceptance
- What is the scope of functional verification

### 5. About Rollback
- What triggers rollback
- Is the source end kept available
- How long does rollback take
- Is data compensation needed after rollback

---

## Eleven. How to Answer This Methodology in an Interview

If the interviewer asks:

- How is migration generally done
- What are full sync and incremental sync
- Why is cut-over needed
- Why is rollback design needed

You can use the following response:

> A standard migration project typically doesn't just do a one-time copy, but is divided into full sync, incremental sync, final cut-over, post-migration validation, and rollback design when needed.  
> Full sync is responsible for moving historical data to the target end first, while incremental sync continuously supplements changes after full sync, which can minimize the final downtime window.  
> The cut-over phase is when the business entry is truly switched from the source to the target end. After the switch, data and function validation are needed to ensure the migration result is usable.  
> Meanwhile, production migration must retain a rollback plan. If the target end fails, the business can be quickly switched back to the source end, reducing change risks.

### One-Sentence Interview Version
> **The core of the migration methodology is: first move existing data, then track incremental changes, switch in a controlled window, and ensure validation and rollback.**

---

## Twelve. Common Misconceptions

### 1. Believing that full sync completion equals migration completion
In reality, as long as the source end is online, the target end will still lag, requiring continuous incremental sync.

### 2. Believing that normal incremental sync guarantees a smooth switch
Incremental sync only tracks data; the real risks often surface during the cut-over.

### 3. Believing that migration tool success means business is fine
Business and data validation are still required.

### 4. Believing that having backups equals having a rollback plan
Backups don't equal quick recovery; rollback must have clear steps and responsible parties.

### 5. Believing that migration can always achieve zero downtime
In most scenarios, sync can approach real-time, but final switching usually still requires short downtime or brief access switching.

---

## Thirteen. Review Keywords for Quick Recall

- Full Sync
- Incremental Sync
- Cut-over
- Validation
- Rollback
- Stop-write
- Switch traffic
- Consistency Validation
- Observation Window
- Rollback Plan

---

## Fourteen. Summary

The true focus of cloud migration isn't the tool name itself, but whether you understand the basic methodology of migration.  
Whether it's host migration, database migration, or object storage migration, the underlying principles can all be summarized as the same core line:

> **Full Sync of existing data, incremental tracking of changes, cut-over for switching, validation for confirmation, and rollback as a safety net.**

These five keywords form the basic closed loop of a migration project and represent the most valuable core understanding for cloud migration roles.