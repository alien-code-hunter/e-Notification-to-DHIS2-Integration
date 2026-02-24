# e-Notification to DHIS2 Data Mapping

This document describes the mapping of all data elements from the e-Notification system to DHIS2 event data elements (all prefixed with "EN_").

| e-Notification Field                | DHIS2 Data Element (EN)         | Notes                                  |
|-------------------------------------|----------------------------------|----------------------------------------|
| patient.firstName                   | EN_PatientName                   | Patient’s full name                    |
| patient.lastName                    | EN_PatientName                   | Combine with firstName if needed       |
| patient.nationalId                  | EN_PatientNationalId (add if needed) | Unique patient identifier         |
| patient.dateOfBirth                 | EN_PatientDOB                    | Date of birth                          |
| patient.sex                         | EN_PatientSex                    | Biological sex                         |
| deathDetails.dateOfDeath            | EN_DateOfDeath                   | Date when death occurred               |
| deathDetails.ageAtDeath             | EN_AgeAtDeath                    | Age at death                           |
| deathDetails.placeOfDeath           | EN_PlaceOfDeath                  | Location of death                      |
| deathDetails.healthFacility         | EN_HealthFacility                | Facility code/name/county              |
| medicalInfo.primaryCauseOfDeath     | EN_PrimaryCauseOfDeath           | ICD-10 code and description            |
| medicalInfo.secondaryCausesOfDeath  | EN_SecondaryCausesOfDeath        | Array of ICD-10 codes/descriptions     |
| medicalInfo.clinicalDiagnosis       | EN_ClinicalDiagnosis (add if needed) | Clinical diagnosis                 |
| medicalInfo.autopsyPerformed        | EN_AutopsyPerformed (add if needed)  | Boolean                               |
| admissionDetails.dateOfAdmission    | EN_DateOfAdmission (add if needed)   | Admission date                        |
| admissionDetails.lengthOfStayDays   | EN_LengthOfStay (add if needed)      | Number of days                        |
| admissionDetails.wardDepartment     | EN_WardDepartment (add if needed)    | Ward/department                       |
| reportingDetails.reportingDate      | EN_ReportingDate                  | Date reported                          |
| reportingDetails.reportedBy         | EN_ReportedBy                     | Person/system reporting                |
| reportingDetails.reportingFacility  | EN_ReportingFacility (add if needed) | Facility name                         |
| reportingDetails.verificationStatus | EN_VerificationStatus             | pending/verified/rejected              |
| additionalInfo.comments             | EN_Comments                       | Additional notes/comments              |
| additionalInfo.phoneNumber          | EN_PhoneNumber (add if needed)       | Next of kin phone                      |
| additionalInfo.nextOfKin            | EN_NextOfKin (add if needed)         | Next of kin name                       |
| externalId                          | EN_ExternalId (add if needed)        | Unique external reference              |

> Ensure your DHIS2 event program includes all EN-prefixed data elements above to capture every e-Notification field.
