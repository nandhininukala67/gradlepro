SELECT
  txn_count,
  COUNT(*) AS wallet_count,
  SUM(txn_count * 5000) AS total_amount
FROM (
  SELECT
    payee_wallet,
    COUNT(*) AS txn_count
  FROM rtsp.token_tranlog
  WHERE payer_wallet = 'SPONSOR_WALLET_ID'
    AND status = 'C'
    AND amount = 5000
  GROUP BY payee_wallet
  HAVING COUNT(*) IN (1, 2, 3)
) sub
GROUP BY txn_count
ORDER BY txn_count;
 
 
 SELECT  
  COUNT(subquery.payee_wallet) AS wallet_count,  
  SUM(subquery.total_value) AS total_value  
FROM (  
  SELECT  
    t.payee_wallet,  
    COUNT(*) AS txn_count,  
    SUM(t.amount) AS total_value  
  FROM rtsp.token_tranlog t  
  WHERE t.status = 'C'  
    AND t.itc = 'Merchant payout.Pool'  
    AND t.amount = 5000  
  GROUP BY t.payee_wallet  
  HAVING COUNT(*) = 1  
) subquery;  




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
















@POST
@Path("/update")
@Consumes(MediaType.APPLICATION_JSON)
@Produces(MediaType.APPLICATION_JSON)
public Response updateKycLimits(List<KycLimit> kycLimits) {
    if (kycLimits == null || kycLimits.isEmpty()) {
        return error(Response.Status.BAD_REQUEST, "No data provided");
    }

    List<KycLimit> updatedRecords = new ArrayList<>(); // 👈 collect updated records

    for (KycLimit dtoLimit : kycLimits) {
        KycLimit existingLimit = kycLimitManager.getByID(dtoLimit.getId());

        if (existingLimit == null) {
            continue; // Skip if not found
        }

        if (!isChanged(existingLimit, dtoLimit)) {
            continue; // Skip if no changes
        }

        // Mark old record inactive
        existingLimit.setStatus("inactive");
        existingLimit.setUpdatedBy(getCurrentUser());
        existingLimit.setUpdatedOn(new Date());
        dbService.saveOrUpdate(existingLimit);

        // Create new record
        KycLimit newLimit = new KycLimit();
        newLimit.setId(UUID.randomUUID().toString()); // or setId(null) if auto-gen
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

        dbService.save(newLimit);

        // Optional: send to external switch
        if (StringUtils.equalsIgnoreCase(newLimit.getLimitType(), KYC_LIMIT_TYPE_NORMAL)) {
            sendToRTSPSwitch(newLimit);
        }

        updatedRecords.add(newLimit); // 👈 Collect the newly created active record
    }

    if (updatedRecords.isEmpty()) {
        return error(Response.Status.NOT_MODIFIED, "No records were updated");
    }

    return Response.ok(new ApiResponseWithData<>(true, "Updated Successfully", updatedRecords))
            .build();
}















private boolean isChanged(KycLimit oldLimit, KycLimit newLimit) {
    if (!Objects.equals(oldLimit.getCapacityLimit(), newLimit.getCapacityLimit())) return true;
    if (!Objects.equals(oldLimit.getPerDayLoadLimit(), newLimit.getPerDayLoadLimit())) return true;
    if (!Objects.equals(oldLimit.getPerDayUnLoadLimit(), newLimit.getPerDayUnLoadLimit())) return true;
    if (!Objects.equals(oldLimit.getPerDayTrfInwardLimit(), newLimit.getPerDayTrfInwardLimit())) return true;
    if (!Objects.equals(oldLimit.getPerDayTfrOutwardLimit(), newLimit.getPerDayTfrOutwardLimit())) return true;
    if (!Objects.equals(oldLimit.getTxnLoadCount(), newLimit.getTxnLoadCount())) return true;
    if (!Objects.equals(oldLimit.getTxnTfrInwardCount(), newLimit.getTxnTfrInwardCount())) return true;
    if (!Objects.equals(oldLimit.getTxnUnloadCount(), newLimit.getTxnUnloadCount())) return true;
    if (!Objects.equals(oldLimit.getTxnTrfOutwardCount(), newLimit.getTxnTrfOutwardCount())) return true;
    if (!Objects.equals(oldLimit.getPerTransaction(), newLimit.getPerTransaction())) return true;
    if (!Objects.equals(oldLimit.getMonthlyTrfOutwardCount(), newLimit.getMonthlyTrfOutwardCount())) return true;
    return false;
}


for (KycLimit dtoLimit : kycLimit) {
    log.info("Processing update for KycLimit ID: {}", dtoLimit.getId());

    KycLimit existingLimit = kycLimitManager.getByID(dtoLimit.getId());
    if (existingLimit == null) {
        log.error("No existing KycLimit found for ID: {}", dtoLimit.getId());
        return error(Response.Status.BAD_REQUEST, "Record not found");
    }

    if (!isChanged(existingLimit, dtoLimit)) {
        log.info("No changes detected for ID: {}", dtoLimit.getId());
        return error(Response.Status.NOT_MODIFIED, "Not Changed");
    }

    existingLimit.setStatus("inactive");
    getDB().saveOrUpdate(existingLimit);

    KycLimit newLimit = new KycLimit();
    // Copy fields (you can also consider using BeanUtils.copyProperties for shorter code)
    newLimit.setType(dtoLimit.getType());
    newLimit.setLimitType(dtoLimit.getLimitType());
    newLimit.setCapacityLimit(dtoLimit.getCapacityLimit());
    newLimit.setPerDayLoadLimit(dtoLimit.getPerDayLoadLimit());
    newLimit.setPerDayUnLoadLimit(dtoLimit.getPerDayUnLoadLimit());
    newLimit.setPerDayTrfInwardLimit(dtoLimit.getPerDayTrfInwardLimit());
    newLimit.setPerDayTfrOutwardLimit(dtoLimit.getPerDayTfrOutwardLimit());
    newLimit.setTxnLoadCount(dtoLimit.getTxnLoadCount());
    newLimit.setTxnTfrInwardCount(dtoLimit.getTxnTfrInwardCount()); // Correct spelling
    newLimit.setTxnUnloadCount(dtoLimit.getTxnUnloadCount());
    newLimit.setTxnTrfOutwardCount(dtoLimit.getTxnTrfOutwardCount());
    newLimit.setPerTransaction(dtoLimit.getPerTransaction());
    newLimit.setMonthlyTrfOutwardCount(dtoLimit.getMonthlyTrfOutwardCount());
    newLimit.setStatus("active");
    newLimit.setCreatedBy(getCurrentUser());

    try {
        getDB().save(newLimit);
        log.info("Successfully saved new KycLimit ID: {}", newLimit.getId());
    } catch (Exception e) {
        log.error("Error saving new KycLimit record", e);
        throw e;
    }

    if (StringUtils.equals(newLimit.getLimitType(), KYC_LIMIT_TYPE_NORMAL)) {
        sendToRTSPSwitch(newLimit);
    }
}













