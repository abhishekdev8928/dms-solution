# DMS Access Control Implementation Guide

## 🎯 Purpose
This document provides a **conceptual framework** for building a clear, maintainable access control system for your Document Management System (DMS). Use this guide to understand the principles and logic flow of a robust permission system.

---

## 📋 Table of Contents
1. [Core Problem Identification](#core-problem)
2. [Solution: Access Chain Pattern](#solution)
3. [Access Control Layers](#layers)
4. [Decision Tree](#decision-tree)
5. [Testing Scenarios](#testing)
6. [Common Pitfalls to Avoid](#pitfalls)
7. [Golden Rules Summary](#golden-rules)

---

## 🔍 Core Problem Identification {#core-problem}

### What's Breaking Your System:
- ❌ Multiple overlapping access paths (direct, inherited, shared)
- ❌ No clear resolution order when rules conflict
- ❌ Mixing Role + ACL + Visibility checks everywhere
- ❌ Duplicate permission logic scattered across codebase
- ❌ No single source of truth for access decisions

### What You Need:
- ✅ **ONE** unified access control mechanism
- ✅ Clear priority order for access checks
- ✅ Separation of concerns (Role vs ACL vs Visibility)
- ✅ Predictable, testable access logic

---

## 🔗 Solution: Access Chain Pattern {#solution}

### Concept
Think of access as a **chain of gates**. User must pass ALL gates:

```
Request → Gate 1 (Role) → Gate 2 (Visibility) → Gate 3 (ACL) → ALLOW
          ↓ DENY         ↓ DENY                ↓ DENY
```

### Access Resolution Priority (First Match Wins)
```
1. Super Admin          → BYPASS all checks → ALLOW
2. Resource Owner       → Check ownership rules → ALLOW
3. Visibility = Private → DENY (unless owner/super admin)
4. Visibility = Public  → ALLOW view only
5. Direct ACL Share     → Check explicit permissions
6. Folder ACL (Inherit) → Check parent permissions
7. Default              → DENY
```

---

## 🗃️ Access Control Layers {#layers}

### Layer 1: ROLE (Capability Layer)
**Question: What CAN this role do in general?**

**Role Capabilities Matrix:**
| Role | VIEW | UPLOAD | DOWNLOAD | DELETE | SHARE | MANAGE |
|------|------|--------|----------|--------|-------|--------|
| SUPER_ADMIN | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| ADMIN | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| DEPT_OWNER | ✅ | ✅ | ✅ | ❌ | ✅ | ❌ |
| MEMBER_BANK | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ |
| GENERAL_USER | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |
| PUBLIC | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |

**Rule:** If role doesn't allow an action, ACL cannot override this. Role acts as a **capability ceiling**.

**Critical Permission Delegation Rule:**
> ⚠️ **A resource owner can only grant permissions that the recipient's role already allows.**
> 
> Example: If you share a file with a GENERAL_USER and give them "UPLOAD" permission via ACL, they still **cannot upload** because their role doesn't have the UPLOAD capability. The ACL only determines WHERE they can use their existing role capabilities, not WHAT new capabilities they gain.

---

### Layer 2: VISIBILITY (Scope Layer)
**Question: Who is ALLOWED to see this resource?**

**Visibility Levels:**

| Visibility | Who Can Access | Notes |
|------------|----------------|-------|
| **PUBLIC** | Everyone | View-only access for all users |
| **PRIVATE** | Owner + Super Admin only | **ACL cannot override** - completely blocked to others |
| **RESTRICTED** | Needs explicit ACL | Must have specific permission granted |

**Rule:** Private files are NEVER accessible by ACL, only by owner/super admin.

---

### Layer 3: ACL (Location Layer)
**Question: WHERE is this user allowed to perform actions?**

**ACL Types:**
- **Direct ACL**: Explicitly granted on specific file/folder
- **Inherited ACL**: Permissions inherited from parent folder

**ACL Matching Criteria:**
- User ID match (specific person)
- Role ID match (all users with that role)
- Department ID match (all users in that department)

**Rule:** ACL defines WHERE you can use your role capabilities, but cannot grant capabilities your role doesn't have.

---

## 🌲 Complete Decision Tree {#decision-tree}

```
┌─────────────────────┐
│ Is Super Admin?     │──YES──> ALLOW (bypass all)
└──────────┬──────────┘
           NO
           ↓
┌─────────────────────┐
│ Does Role Allow     │──NO──> DENY
│ This Action?        │
└──────────┬──────────┘
          YES
           ↓
┌─────────────────────┐
│ Is Resource Owner?  │──YES──> ALLOW
└──────────┬──────────┘
           NO
           ↓
┌─────────────────────┐
│ Visibility=PRIVATE? │──YES──> DENY
└──────────┬──────────┘
           NO
           ↓
┌─────────────────────┐
│ Visibility=PUBLIC?  │──YES──> ALLOW (View only)
└──────────┬──────────┘
           NO (RESTRICTED)
           ↓
┌─────────────────────┐
│ Has Direct ACL for  │──YES──> ALLOW
│ This Resource?      │
└──────────┬──────────┘
           NO
           ↓
┌─────────────────────┐
│ Has Folder ACL      │──YES──> ALLOW
│ (Inherited)?        │
└──────────┬──────────┘
           NO
           ↓
         DENY
```

---

## ✅ Testing Scenarios {#testing}

### Test Scenarios (MUST PASS ALL)

#### Scenario 1: Super Admin Access
- [ ] Super admin can VIEW any file (public/private/restricted)
- [ ] Super admin can DOWNLOAD any file
- [ ] Super admin can DELETE any file
- [ ] Super admin can UPLOAD to any folder

#### Scenario 2: Owner Access
- [ ] Owner can VIEW their own private file
- [ ] Owner can DELETE their own file
- [ ] Owner can SHARE their own file
- [ ] Owner can modify file visibility

#### Scenario 3: Private File Protection
- [ ] Non-owner CANNOT view private file (even with folder ACL)
- [ ] Non-owner CANNOT download private file
- [ ] Department owner CANNOT access user's private file in their folder
- [ ] Admin CANNOT access private file (unless super admin)

#### Scenario 4: Public File Access
- [ ] Anyone can VIEW public file
- [ ] Anyone can DOWNLOAD public file
- [ ] Only owner/super admin can DELETE public file
- [ ] Only owner/super admin can change visibility

#### Scenario 5: ACL Direct Share & Role Limitations
- [ ] User with direct file ACL can VIEW restricted file
- [ ] User with folder ACL can VIEW restricted files in that folder
- [ ] User WITHOUT ACL cannot access restricted file
- [ ] **CRITICAL**: Shared permission respects role limits (General User can't UPLOAD even if ACL grants it, because their role lacks UPLOAD capability)
- [ ] **CRITICAL**: Owner sharing with GENERAL_USER + "DELETE" permission → User still cannot DELETE (role limitation)

#### Scenario 6: Folder Inheritance
- [ ] User with folder VIEW permission can view all restricted files inside
- [ ] User with folder UPLOAD permission can upload to folder (if role allows UPLOAD)
- [ ] Folder permission does NOT grant access to private files inside
- [ ] Nested folder inherits parent permissions correctly

#### Scenario 7: Role as Capability Ceiling
- [ ] General User CANNOT upload even with folder ACL granting upload
- [ ] General User CAN view and download if ACL allows (role permits these)
- [ ] Member Bank User CAN upload if ACL allows (role permits upload)
- [ ] **Role acts as ceiling**: ACL cannot override role restrictions

---

## ⚠️ Common Pitfalls to Avoid {#pitfalls}

### ❌ DON'T DO THIS:

1. **Allowing ACL to override role capabilities**
   - Wrong: "User has ACL permission for UPLOAD, so they can upload"
   - Right: "User has ACL permission AND role allows UPLOAD, so they can upload"

2. **Treating folder owner as file owner**
   - Wrong: "User owns the folder, so they can access all files inside"
   - Right: "Folder ownership doesn't grant automatic access to private files inside"

3. **Ignoring visibility when checking ACL**
   - Wrong: "User has ACL permission, grant access"
   - Right: "Check visibility first - if PRIVATE, deny even with ACL"

4. **Granting permissions the recipient's role doesn't support**
   - Wrong: "I'll share this with GENERAL_USER and give them DELETE permission"
   - Right: "GENERAL_USER role can't DELETE, so this ACL permission is meaningless"

5. **Multiple conflicting permission checks without clear priority**
   - Wrong: Checking ACL, then ownership, then visibility in random order
   - Right: Follow the decision tree: Admin → Owner → Visibility → ACL

---

### ✅ DO THIS INSTEAD:

1. **Always check role first, then ACL**
   - Role defines the ceiling of what's possible
   - ACL only narrows WHERE that capability can be used

2. **Respect ownership boundaries**
   - Folder ownership ≠ file access rights
   - Each resource has its own owner

3. **Check visibility before ACL**
   - PRIVATE blocks everyone except owner/super admin
   - No amount of ACL permissions can override PRIVATE

4. **Validate role capabilities when granting permissions**
   - Before adding ACL entry, verify recipient's role supports that action
   - Warn users when granting meaningless permissions

5. **Follow clear priority order consistently**
   - Super Admin → Owner → Visibility → ACL
   - Never deviate from this sequence

---

## 📊 Golden Rules Summary {#golden-rules}

| Rule | Description |
|------|-------------|
| **#1** | Role defines WHAT you can do (capability ceiling) - ACL cannot exceed this |
| **#2** | ACL defines WHERE you can do it (location boundary within role limits) |
| **#3** | Visibility defines WHO can see it (scope gate) |
| **#4** | Access granted ONLY if Role + ACL + Visibility all allow |
| **#5** | Super Admin bypasses all checks |
| **#6** | Owner = God of their resource (except super admin exists) |
| **#7** | Private = Owner + Super Admin ONLY (no ACL override possible) |
| **#8** | Folder owner ≠ File access (ownership doesn't inherit) |
| **#9** | ONE centralized access control mechanism for ALL decisions |
| **#10** | Check priority: Admin → Owner → Visibility → ACL (never change order) |
| **#11** | **Resource owners can only grant permissions the recipient's role allows** |

---

## 🔗 Quick Reference

### Action Types
- **VIEW** - Can see the resource exists and its metadata
- **DOWNLOAD** - Can download/read the file content
- **UPLOAD** - Can create new files in folder
- **DELETE** - Can remove the resource
- **SHARE** - Can modify ACL and share with others
- **MANAGE** - Can change settings, move, rename

### Visibility Types
- **PUBLIC** - Anyone can view (read-only for non-owners)
- **PRIVATE** - Owner + Super Admin only (ACL cannot override)
- **RESTRICTED** - Need explicit ACL grant

### Access Decision Flow
```
User Request → Check Super Admin → Check Role Capability → 
Check Ownership → Check Visibility → Check ACL → Decision
```

---

**END OF CONCEPTUAL GUIDE**

*Use this document as a reference when designing or reviewing your access control architecture.*
