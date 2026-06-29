# Vendor Form - Comprehensive Test Plan

## Overview
This document provides a thorough testing checklist for the refactored 5-step vendor form.

**Test Date**: 2026-01-29
**Tester**: [Your Name]
**Environment**: Development

---

## 🎯 Test Objectives

1. Verify all 5 steps work correctly
2. Validate form data persistence and auto-save
3. Test document upload functionality with drag-and-drop
4. Ensure proper navigation between steps
5. Verify form submission and data integrity
6. Test error handling and validation
7. Confirm UI/UX matches Deal Wizard design

---

## ✅ Pre-Test Setup

- [ ] Backend running on `http://localhost:5002`
- [ ] Frontend running on `http://localhost:5001`
- [ ] Database accessible and seeded
- [ ] Browser developer tools open (Console + Network tabs)
- [ ] Clear localStorage before starting fresh tests

```javascript
// Clear localStorage in browser console
localStorage.clear();
```

---

## 📋 Test Cases

### **Test Suite 1: Visual Design & Styling**

#### TC1.1: Container Styling
- [ ] **Navigate to**: `/vendor-management/add-vendor`
- [ ] Verify sidebar has white background
- [ ] Verify main content area has `bg-gray-100` (light gray)
- [ ] Verify form cards have `shadow-sm` instead of borders
- [ ] Verify max-width is `max-w-4xl` (wider than before)
- [ ] Verify header is sticky at top with proper padding
- [ ] Verify back button navigation to `/vendor-management` works

**Expected**: Form should match Deal Wizard visual design

---

#### TC1.2: Step Progress Indicator
- [ ] Verify VerticalStepProgress component displays on left sidebar
- [ ] Verify 5 steps are shown (not 7)
- [ ] Verify current step is highlighted
- [ ] Verify completed steps show checkmark
- [ ] Verify click on completed steps navigates correctly
- [ ] Verify click on future steps is disabled

**Expected Steps**:
1. Basic & Company Info
2. Location Details
3. Financial & Banking
4. Contact & Documents
5. Review & Submit

---

### **Test Suite 2: Step 1 - Basic & Company Info**

#### TC2.1: Basic Information Section
- [ ] Navigate to Step 1
- [ ] Verify section header: "Contact Details"
- [ ] Enter name: "Test Vendor"
- [ ] Enter phone: "1234567890" (10 digits)
- [ ] Enter email: "test@vendor.com"
- [ ] Verify validation on empty fields
- [ ] Verify phone validation (must be 10 digits)
- [ ] Verify email validation (valid format)

**Test Invalid Data**:
- [ ] Phone: "123" → Should show error: "Phone number must be 10 digits"
- [ ] Email: "invalid-email" → Should show error: "Please enter a valid email address"
- [ ] Empty name → Should show error: "Name is required"

---

#### TC2.2: Company Information Section
- [ ] Verify section header: "Company Information"
- [ ] Verify divider line separates contact and company sections
- [ ] Enter company name: "Test Company LLC"
- [ ] Select establishment date: "2020-01-01"
- [ ] Select nature: "Domestic"
- [ ] Verify all fields are properly labeled
- [ ] Click "Next" button

**Expected**: Should navigate to Step 2, data should be saved

---

### **Test Suite 3: Step 2 - Location Details**

#### TC3.1: Address Fields
- [ ] Verify you're on Step 2
- [ ] Verify section header: "Company Address"
- [ ] Enter address: "123 Main Street, Suite 100"
- [ ] Enter city: "Mumbai"
- [ ] Enter state: "Maharashtra"
- [ ] Enter zip code: "400001" (5-6 digits)
- [ ] Select country: "India"

**Test Validations**:
- [ ] All fields required → Should show errors
- [ ] Invalid zip code (e.g., "12") → Should show error
- [ ] Click "Previous" → Should go back to Step 1 with data preserved
- [ ] Click "Next" → Should advance to Step 3

---

### **Test Suite 4: Step 3 - Financial & Banking**

#### TC4.1: Currency Selection
- [ ] Verify section header: "Currency Preference"
- [ ] Select currency: "INR"
- [ ] Verify dropdown shows: INR, USD, EUR, GBP, AUD
- [ ] Change selection to "USD"
- [ ] Verify selection updates correctly

---

#### TC4.2: Banking Information
- [ ] Verify section header: "Banking Information"
- [ ] Enter bank name: "HDFC Bank"
- [ ] Enter beneficiary name: "Test Company LLC"
- [ ] Enter account number: "12345678901234"
- [ ] Enter IBAN: "HDFC0001234"
- [ ] Enter SWIFT code: "HDFCINBB"
- [ ] Enter IFSC code: "HDFC0001234"
- [ ] Enter bank account type: "Current"
- [ ] Enter bank address: "Bandra, Mumbai"

---

#### TC4.3: Cancelled Cheque Upload
- [ ] Verify file upload label: "Upload Cancelled Cheque (Optional)"
- [ ] Click file input and select an image (.jpg/.png)
- [ ] Verify file preview appears (thumbnail with border)
- [ ] Upload a PDF file
- [ ] Verify file size and format displayed
- [ ] Test max file size (10MB limit)
- [ ] Upload file >10MB → Should show error
- [ ] Click "Next" → Should advance to Step 4

---

### **Test Suite 5: Step 4 - Contact & Documents**

#### TC5.1: Point of Contact Fields
- [ ] Verify section header: "Point of Contact"
- [ ] Enter contact person: "John Doe"
- [ ] Enter designation: "Procurement Manager"
- [ ] Enter email: "john@testcompany.com"
- [ ] Enter phone: "9876543210"
- [ ] Enter website: "https://testcompany.com"

**Test Validations**:
- [ ] Invalid email format → Should show error
- [ ] Invalid phone (not 10 digits) → Should show error
- [ ] Website is optional → Should work without it

---

#### TC5.2: Drag-and-Drop File Upload
**Visual Elements**:
- [ ] Verify drag-drop zone is visible
- [ ] Verify upload icon (SVG) is centered
- [ ] Verify text: "Drag and drop files here, or click to browse"
- [ ] Verify supported formats: "PDF, DOC, DOCX, JPG, PNG (Max 10MB per file)"

**Test Drag Events**:
- [ ] Drag a file over the zone
- [ ] Verify border changes to blue (isDragging state)
- [ ] Verify background changes to blue-50
- [ ] Verify text changes to "Drop files here"
- [ ] Drop the file
- [ ] Verify file appears in document list below

---

#### TC5.3: File Upload via Click
- [ ] Click anywhere in the drag-drop zone
- [ ] Verify file picker opens
- [ ] Select 1 PDF file
- [ ] Verify file appears in list with:
  - [ ] Correct file icon (red for PDF)
  - [ ] File name
  - [ ] File size in MB
  - [ ] File type (uppercase extension)
  - [ ] Remove button (trash icon)

---

#### TC5.4: Multiple File Upload
- [ ] Upload 3 different files (PDF, DOC, JPG)
- [ ] Verify all 3 files appear in list
- [ ] Verify correct icons for each file type:
  - [ ] PDF → Red PDF icon
  - [ ] DOC/DOCX → Blue document icon
  - [ ] JPG/PNG → Green image icon
- [ ] Verify "Uploaded Documents (3)" count
- [ ] Verify "Clear All" button appears

---

#### TC5.5: File Validation
**Test Invalid File Types**:
- [ ] Try uploading .exe file → Should show error
- [ ] Try uploading .zip file → Should show error
- [ ] Verify error message: "Invalid file type. Only PDF, DOC, DOCX, JPG, PNG are allowed."
- [ ] Verify toast notification appears

**Test File Size Limit**:
- [ ] Upload file > 10MB → Should show error
- [ ] Verify error message: "File size exceeds 10MB limit."

**Test Duplicate Files**:
- [ ] Upload same file twice
- [ ] Verify error: "File already added."
- [ ] Verify file is not duplicated in list

---

#### TC5.6: File Management
- [ ] Click remove button on one file
- [ ] Verify file is removed from list
- [ ] Verify count updates
- [ ] Verify toast: "File removed"
- [ ] Click "Clear All" button
- [ ] Verify all files removed
- [ ] Verify toast: "All files removed"
- [ ] Verify upload zone still visible

---

#### TC5.7: Error Display
- [ ] Upload an invalid file to trigger error
- [ ] Verify red error banner appears below upload zone
- [ ] Verify error icon (red X) appears
- [ ] Verify error message is readable
- [ ] Click X button on error banner
- [ ] Verify error disappears

---

### **Test Suite 6: Step 5 - Review & Submit**

#### TC6.1: Review Section Layout
- [ ] Click "Next" from Step 4
- [ ] Verify you're on Step 5
- [ ] Verify header: "Review & Submit"
- [ ] Verify subtitle: "Please review all information before submitting"
- [ ] Verify 4 card sections are visible (white cards with borders)

---

#### TC6.2: Review Card 1 - Basic & Company Info
- [ ] Verify card header: "Step 1: Basic & Company Information"
- [ ] Verify "Contact Details" subsection
- [ ] Verify name is displayed: "Test Vendor"
- [ ] Verify email is displayed: "test@vendor.com"
- [ ] Verify phone is displayed: "1234567890"
- [ ] Verify divider line between subsections
- [ ] Verify "Company Details" subsection
- [ ] Verify company name: "Test Company LLC"
- [ ] Verify establishment date is formatted correctly
- [ ] Verify nature of business: "Domestic"

---

#### TC6.3: Review Card 2 - Location Details
- [ ] Verify card header: "Step 2: Location Details"
- [ ] Verify address: "123 Main Street, Suite 100"
- [ ] Verify city: "Mumbai"
- [ ] Verify state: "Maharashtra"
- [ ] Verify zip code: "400001"
- [ ] Verify country: "India"
- [ ] All fields should show "N/A" if empty

---

#### TC6.4: Review Card 3 - Financial & Banking
- [ ] Verify card header: "Step 3: Financial & Banking"
- [ ] Verify "Currency" subsection
- [ ] Verify currency: "INR" or "USD"
- [ ] Verify divider line
- [ ] Verify "Banking Information" subsection
- [ ] Verify all bank details are displayed
- [ ] Verify cancelled cheque thumbnail if uploaded
- [ ] Verify image has border and rounded corners

---

#### TC6.5: Review Card 4 - Contact & Documents
- [ ] Verify card header: "Step 4: Contact & Documents"
- [ ] Verify "Point of Contact" subsection
- [ ] Verify contact person: "John Doe"
- [ ] Verify all contact fields
- [ ] Verify "Documents" subsection
- [ ] Verify text: "Supporting documents will be displayed here once uploaded."

---

#### TC6.6: Submit Form
- [ ] Click "Complete" button
- [ ] Verify button has blue background (bg-blue-600)
- [ ] Verify button shows hover effect (bg-blue-700)
- [ ] Verify navigation to `/vendor-management`
- [ ] Verify success toast: "Vendor created successfully"
- [ ] Verify auto-saved draft is cleared from localStorage

---

### **Test Suite 7: Auto-Save Functionality**

#### TC7.1: Auto-Save Indicator
- [ ] Start fresh (clear localStorage)
- [ ] Navigate to Step 1
- [ ] Fill in name field
- [ ] Wait 30 seconds
- [ ] Verify auto-save indicator appears in header
- [ ] Verify text: "Saving..." (while saving)
- [ ] Verify icon changes to green checkmark with "Saved just now"

---

#### TC7.2: Save Interval
- [ ] Fill in more fields
- [ ] Wait 30 seconds
- [ ] Verify indicator updates: "Saved just now"
- [ ] Wait 2 minutes
- [ ] Verify indicator updates: "Saved 2 min ago"
- [ ] Navigate to Step 2
- [ ] Make changes
- [ ] Wait 30 seconds
- [ ] Verify auto-save still works on Step 2

---

#### TC7.3: Draft Loading
- [ ] Fill in fields on Step 1 and Step 2
- [ ] Wait for auto-save
- [ ] Close browser tab (or refresh page)
- [ ] Navigate back to `/vendor-management/add-vendor`
- [ ] Verify toast: "Draft loaded from previous session"
- [ ] Verify form data is restored
- [ ] Verify current step is restored
- [ ] Verify last saved timestamp is shown

---

#### TC7.4: Draft Expiration
- [ ] Manually edit localStorage:
  ```javascript
  const key = 'vendor-form-draft-new';
  const data = JSON.parse(localStorage.getItem(key));
  data.timestamp = '2020-01-01T00:00:00.000Z'; // Old date
  localStorage.setItem(key, JSON.stringify(data));
  ```
- [ ] Refresh page
- [ ] Verify draft is NOT loaded (older than 7 days)
- [ ] Verify form starts fresh

---

#### TC7.5: No Auto-Save on Review
- [ ] Navigate to Step 5 (Review)
- [ ] Verify auto-save indicator is NOT visible
- [ ] Verify no auto-save activity (check localStorage)
- [ ] This prevents unnecessary saves on review page

---

### **Test Suite 8: Navigation & Data Persistence**

#### TC8.1: Forward Navigation
- [ ] Fill Step 1, click Next
- [ ] Fill Step 2, click Next
- [ ] Fill Step 3, click Next
- [ ] Fill Step 4, click Next
- [ ] Verify arrive at Step 5
- [ ] Verify URL updates: `?step=5`

---

#### TC8.2: Backward Navigation
- [ ] From Step 5, click Previous
- [ ] Verify Step 4 loads with data intact
- [ ] Click Previous again
- [ ] Verify Step 3 loads with data intact
- [ ] Continue to Step 1
- [ ] Verify all data is preserved

---

#### TC8.3: Sidebar Step Navigation
- [ ] Fill Steps 1-3
- [ ] Click on "Step 1" in sidebar
- [ ] Verify navigate to Step 1
- [ ] Verify data is still there
- [ ] Try clicking "Step 4" (future step)
- [ ] Verify click is disabled/ignored
- [ ] Only allow backward navigation

---

#### TC8.4: Browser Back/Forward
- [ ] Use browser back button from Step 3
- [ ] Verify navigate to Step 2
- [ ] Verify URL updates
- [ ] Use browser forward button
- [ ] Verify navigate back to Step 3
- [ ] Verify data is preserved

---

#### TC8.5: URL Direct Access
- [ ] Manually navigate to `?step=3`
- [ ] Verify Step 3 loads
- [ ] Manually navigate to `?step=10` (invalid)
- [ ] Verify defaults to Step 1
- [ ] Manually navigate to `?step=0` (invalid)
- [ ] Verify defaults to Step 1

---

### **Test Suite 9: Form Validation**

#### TC9.1: Step 1 Validation
- [ ] Leave all fields empty
- [ ] Click "Next"
- [ ] Verify error messages appear:
  - [ ] "Name is required"
  - [ ] "Phone number is required"
  - [ ] "Email is required"
  - [ ] "Company name is required"
  - [ ] "Establishment date is required"
  - [ ] "Type of business is required"

---

#### TC9.2: Step 2 Validation
- [ ] Leave all fields empty
- [ ] Click "Next"
- [ ] Verify error messages:
  - [ ] "Address is required"
  - [ ] "City is required"
  - [ ] "State is required"
  - [ ] "Zip code is required"
  - [ ] "Country is required"

---

#### TC9.3: Step 3 Validation
- [ ] Leave currency unselected
- [ ] Click "Next"
- [ ] Verify error: "Currency type is required"
- [ ] Bank fields are optional, should not block

---

#### TC9.4: Step 4 Validation
- [ ] All fields are optional
- [ ] Click "Next" with empty form
- [ ] Verify can proceed to Step 5
- [ ] Go back and test email validation
- [ ] Enter "invalid-email"
- [ ] Verify error message

---

### **Test Suite 10: Error Handling**

#### TC10.1: Network Errors
**Simulate API Failure**:
- [ ] Open browser DevTools → Network tab
- [ ] Set throttling to "Offline"
- [ ] Fill Step 1 and click "Next"
- [ ] Verify error toast appears
- [ ] Verify user-friendly error message
- [ ] Re-enable network
- [ ] Retry submission
- [ ] Verify works correctly

---

#### TC10.2: API Timeout
- [ ] Use Network tab "Slow 3G" throttling
- [ ] Submit form data
- [ ] Verify loading state shows
- [ ] Verify timeout handling if API is slow
- [ ] Verify appropriate error message

---

#### TC10.3: Invalid API Response
**Test with Mock Data**:
- [ ] Submit form
- [ ] If API returns error status (400/500)
- [ ] Verify error toast appears
- [ ] Verify error message is clear
- [ ] Verify form remains filled (data not lost)

---

### **Test Suite 11: Responsive Design**

#### TC11.1: Desktop (1920px)
- [ ] Test on desktop resolution
- [ ] Verify sidebar is 320px (w-80)
- [ ] Verify content area max-width: 1024px (max-w-4xl)
- [ ] Verify form fields in 2 columns (md:grid-cols-2)
- [ ] Verify all spacing looks correct

---

#### TC11.2: Tablet (768px)
- [ ] Resize browser to 768px width
- [ ] Verify layout still works
- [ ] Verify sidebar remains visible
- [ ] Verify form fields adjust to single column
- [ ] Verify mobile responsiveness

---

#### TC11.3: Mobile (375px)
- [ ] Resize to mobile width
- [ ] Verify sidebar behavior (should still show)
- [ ] Verify form is usable
- [ ] Verify buttons are touch-friendly
- [ ] Verify text is readable

---

### **Test Suite 12: Browser Compatibility**

#### TC12.1: Chrome
- [ ] Run all tests in Chrome
- [ ] Verify drag-and-drop works
- [ ] Verify file upload works
- [ ] Verify localStorage works

#### TC12.2: Firefox
- [ ] Run critical tests in Firefox
- [ ] Verify core functionality works

#### TC12.3: Safari
- [ ] Run critical tests in Safari
- [ ] Verify drag-and-drop works
- [ ] Verify all features work

---

### **Test Suite 13: Edge Cases**

#### TC13.1: Very Long Text
- [ ] Enter 500 character text in address field
- [ ] Verify field handles it
- [ ] Verify no UI breaking
- [ ] Verify truncation on review page if needed

---

#### TC13.2: Special Characters
- [ ] Enter name: "O'Connor & Sons LLC"
- [ ] Enter email: "test+tag@domain.co.uk"
- [ ] Verify accepts special characters
- [ ] Verify displays correctly on review

---

#### TC13.3: Multiple Rapid Clicks
- [ ] Fill Step 1
- [ ] Click "Next" button rapidly 5 times
- [ ] Verify only advances once
- [ ] Verify no duplicate submissions

---

#### TC13.4: Incomplete Form Abandonment
- [ ] Fill Step 1 and Step 2
- [ ] Navigate away to `/vendor-management`
- [ ] Verify draft is auto-saved
- [ ] Come back later
- [ ] Verify can resume from where left off

---

### **Test Suite 14: Performance**

#### TC14.1: Form Load Time
- [ ] Clear cache
- [ ] Navigate to `/vendor-management/add-vendor`
- [ ] Measure time to interactive
- [ ] Should be < 2 seconds

---

#### TC14.2: Step Navigation Speed
- [ ] Navigate between steps
- [ ] Verify instant navigation (no lag)
- [ ] Verify no flashing or layout shifts

---

#### TC14.3: Auto-Save Performance
- [ ] Type rapidly in multiple fields
- [ ] Verify auto-save doesn't cause UI lag
- [ ] Verify typing is not interrupted

---

#### TC14.4: File Upload Performance
- [ ] Upload 5 files at once
- [ ] Verify all files process correctly
- [ ] Verify no UI freeze
- [ ] Verify smooth user experience

---

## 🐛 Bug Tracking Template

Use this template to report any bugs found:

```markdown
### Bug #[NUMBER]

**Title**: [Short description]

**Severity**: Critical / High / Medium / Low

**Steps to Reproduce**:
1. Step 1
2. Step 2
3. Step 3

**Expected Behavior**:
[What should happen]

**Actual Behavior**:
[What actually happened]

**Screenshot**:
[Attach screenshot if applicable]

**Browser/Environment**:
- Browser: Chrome 120.x
- OS: macOS Sonoma
- Screen Resolution: 1920x1080

**Console Errors**:
```
[Paste any console errors here]
```

**Additional Notes**:
[Any other relevant information]
```

---

## ✅ Test Summary Report

**Test Execution Date**: _____________

**Total Test Cases**: 140+

**Passed**: _____ / _____

**Failed**: _____ / _____

**Blocked**: _____ / _____

**Pass Rate**: _____%

### Critical Issues Found:
1. [Issue 1]
2. [Issue 2]

### Non-Critical Issues:
1. [Issue 1]
2. [Issue 2]

### Recommendations:
1. [Recommendation 1]
2. [Recommendation 2]

---

## 🎉 Sign-Off

**Tested By**: _______________________

**Date**: _______________________

**Status**: ☐ PASSED   ☐ FAILED   ☐ PASSED WITH ISSUES

**Notes**:
_______________________________________________________________
_______________________________________________________________
_______________________________________________________________
