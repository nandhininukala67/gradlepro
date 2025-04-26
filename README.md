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







