for (KycLimit dtoLimit : kycLimit) {
    // Fetch existing record from DB
    KycLimit existingLimit = kycLimitManager.getByID(dtoLimit.getId());

    // If no changes, return "Not Changed" status
    if (!isChanged(existingLimit, dtoLimit)) {
        return error(Response.Status.NOT_MODIFIED, "Not Changed");
    }

    // Mark the existing record as inactive
    existingLimit.setStatus("inactive");
    getDB().saveOrUpdate(existingLimit);

    // Create a new record with updated data
    KycLimit newLimit = new KycLimit();
    newLimit.setType(dtoLimit.getType());
    newLimit.setLimitType(dtoLimit.getLimitType());
    newLimit.setCapacityLimit(dtoLimit.getCapacityLimit());
    newLimit.setPerDayLoadLimit(dtoLimit.getPerDayLoadLimit());
    newLimit.setPerDayUnLoadLimit(dtoLimit.getPerDayUnLoadLimit());
    newLimit.setPerDayTrfInwardLimit(dtoLimit.getPerDayTrfInwardLimit());
    newLimit.setPerDayTfrOutwardLimit(dtoLimit.getPerDayTfrOutwardLimit());
    newLimit.setTxnLoadCount(dtoLimit.getTxnLoadCount());
    newLimit.setTxnLTfrInwardCount(dtoLimit.getTxnLTfrInwardCount());
    newLimit.setTxnUnloadCount(dtoLimit.getTxnUnloadCount());
    newLimit.setTxnTrfOutwardCount(dtoLimit.getTxnTrfOutwardCount());
    newLimit.setPerTransaction(dtoLimit.getPerTransaction());
    newLimit.setMonthlyTrfOutwardCount(dtoLimit.getMonthlyTrfOutwardCount());
    newLimit.setStatus("active"); // mark the new record as active
    newLimit.setCreatedBy(getCurrentUser()); // Optional: If you track user who made the changes

    // Save the new record
    getDB().save(newLimit);

    // Optionally send the new data to a switch or external system
    if (StringUtils.equals(newLimit.getLimitType(), KYC_LIMIT_TYPE_NORMAL)) {
        sendToRTSPSwitch(newLimit);
    }
}



ngOnInit(): void {
  this.kycService.getDetails().subscribe(res => {
    if (res && res.is_success) {
      // Filter the data to only show records with 'active' status
      const activeRecords = res.data.filter((record: any) => record.status === 'active');
      
      // Set the filtered records to the data source for the table
      this.dataSource = new MatTableDataSource(activeRecords);
      this.dsSource = new MatTableDataSource(JSON.parse(JSON.stringify(activeRecords))); // Clone the filtered data
    }
  });
}

onSubmit(event: any) {
  // Submit only the records that have been edited
  const updatedData = this.dataSource?.filteredData?.filter((x: any) => x.isChecked);
  const payload = updatedData.map(({ entityName, isChecked, ...rest }) => {
    rest.perDayTrfInwardLimit = rest.perDayLoadLimit;
    rest.txnLtfrInwardCount = rest.txnLoadCount;
    rest.perDayTrfOutwardLimit = rest.perDayUnLoadLimit;
    rest.txnTrfOutwardCount = rest.txnUnloadCount;
    
    return rest;
  });

  if (payload.length > 0) {
    this.kycService.updateKyc(payload).subscribe(
      data => {
        if (data && data.success) {
          this.isEditable = false;
          this.isEditChecked = false;
          this.dataSource.filteredData.forEach((x: any) => x.isChecked = false);
          this.dsSource.data = JSON.parse(JSON.stringify(this.dataSource.data)); // Reset the data
        } else if (data && data.message) {
          this.toastr.error(data.message);
        } else {
          this.toastr.error('Server error');
        }
      }
    );
  }
}


















public Response updateKycLimits(List<KycLimit> kycLimitList) {
    if (kycLimitList == null || kycLimitList.isEmpty()) {
        return error(Response.Status.BAD_REQUEST, "Input list is empty");
    }

    for (KycLimit dtoLimit : kycLimitList) {
        try {
            if (dtoLimit.getId() == null) {
                return error(Response.Status.BAD_REQUEST, "Missing ID for update");
            }

            KycLimit existingLimit = kycLimitManager.getByID(dtoLimit.getId());

            if (existingLimit == null) {
                return error(Response.Status.NOT_FOUND, "No record found for ID: " + dtoLimit.getId());
            }

            if (!isChanged(existingLimit, dtoLimit)) {
                return error(Response.Status.NOT_MODIFIED, "No changes detected for ID: " + dtoLimit.getId());
            }

            // Step 1: Mark old record as inactive
            existingLimit.setStatus("inactive");
            existingLimit.setUpdatedOn(new Date()); // if you track update timestamps
            getDB().saveOrUpdate(existingLimit);

            // Step 2: Insert new active record
            KycLimit newLimit = new KycLimit();
            newLimit.setType(dtoLimit.getType());
            newLimit.setLimitType(dtoLimit.getLimitType());
            newLimit.setCapacityLimit(dtoLimit.getCapacityLimit());
            newLimit.setPerDayLoadLimit(dtoLimit.getPerDayLoadLimit());
            newLimit.setPerDayUnLoadLimit(dtoLimit.getPerDayUnLoadLimit());
            newLimit.setPerDayTrfInwardLimit(dtoLimit.getPerDayTrfInwardLimit());
            newLimit.setPerDayTfrOutwardLimit(dtoLimit.getPerDayTfrOutwardLimit());
            newLimit.setTxnLoadCount(dtoLimit.getTxnLoadCount());
            newLimit.setTxnLTfrInwardCount(dtoLimit.getTxnLTfrInwardCount());
            newLimit.setTxnUnloadCount(dtoLimit.getTxnUnloadCount());
            newLimit.setTxnTrfOutwardCount(dtoLimit.getTxnTrfOutwardCount());
            newLimit.setPerTransaction(dtoLimit.getPerTransaction());
            newLimit.setMonthlyTrfOutwardCount(dtoLimit.getMonthlyTrfOutwardCount());
            newLimit.setStatus("active");
            newLimit.setCreatedBy(getCurrentUser());
            newLimit.setCreatedOn(new Date());

            getDB().save(newLimit);

            // Step 3: Optionally notify external systems
            if (StringUtils.equals(newLimit.getLimitType(), KYC_LIMIT_TYPE_NORMAL)) {
                sendToRTSPSwitch(newLimit);
            }

        } catch (Exception e) {
            // Basic logging
            log.error("Error updating KycLimit for ID: " + dtoLimit.getId(), e);
            return error(Response.Status.INTERNAL_SERVER_ERROR, "Error updating KYC Limit: " + e.getMessage());
        }
    }

    return success("KYC limits updated successfully");
}

















onSubmit(event: any) {
  const updatedData = this.dataSource?.filteredData?.filter((x: any) => x.isChecked);

  const payload = updatedData.map((record: any) => ({
    id: record.id, // **IMPORTANT** must send existing id
    type: record.type,
    limitType: record.limitType,
    capacityLimit: record.capacityLimit,
    perDayLoadLimit: record.perDayLoadLimit,
    perDayUnLoadLimit: record.perDayUnLoadLimit,
    perDayTrfInwardLimit: record.perDayLoadLimit,         // Mapping correction
    perDayTrfOutwardLimit: record.perDayUnLoadLimit,
    txnLoadCount: record.txnLoadCount,
    txnLTfrInwardCount: record.txnLoadCount,               // Mapping correction
    txnUnloadCount: record.txnUnloadCount,
    txnTrfOutwardCount: record.txnUnloadCount,
    perTransaction: record.perTransaction,
    monthlyTrfOutwardCount: record.monthlyTrfOutwardCount,
    status: 'active',
  }));

  console.log('Payload:', payload);

  if (payload.length > 0) {
    this.kycService.updateKyc(payload).subscribe(
      data => {
        if (data && data.success) {
          this.toastr.success('KYC limits updated successfully');
          this.isEditable = false;
          this.isEditChecked = false;
          this.fetchKycDetails(); // Refresh the updated data from backend
        } else if (data && data.message) {
          this.toastr.error(data.message);
        } else {
          this.toastr.error('Server error during update');
        }
      },
      error => {
        this.toastr.error('Network error during update');
      }
    );
  } else {
    this.toastr.warning('No records selected for update');
  }
}





fetchKycDetails() {
  this.kycService.getDetails().subscribe(res => {
    if (res && res.is_success) {
      const activeRecords = res.data.filter((record: any) => record.status === 'active');
      this.dataSource = new MatTableDataSource(activeRecords);
      this.dsSource = new MatTableDataSource(JSON.parse(JSON.stringify(activeRecords))); // Deep clone
    } else {
      this.toastr.error('Failed to load KYC details');
    }
  });
}
















public class ApiResponseWithData<T> {
    private boolean success;
    private String message;
    private T data;

    public ApiResponseWithData(boolean success, String message, T data) {
        this.success = success;
        this.message = message;
        this.data = data;
    }

    // Getters and setters
}


