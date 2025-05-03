import { ChangeDetectorRef, Component, OnInit } from '@angular/core';
import { MatCheckboxChange } from '@angular/material/checkbox';
import { MatDialog } from '@angular/material/dialog';
import { MatTableDataSource } from '@angular/material/table';
import { ToastrService } from 'ngx-toastr';
import { KycService } from 'src/app/Services/kyc-management.service';

@Component({
  selector: 'app-kyc-management',
  templateUrl: './kyc-management.component.html',
  styleUrls: ['./kyc-management.component.scss']
})
export class KycManagementComponent implements OnInit {
  colDef: any[] = [
    { name: 'type', type: 'readable', readonly: true },
    { name: 'capacityLimit', type: 'number' },
    { name: 'perDayLoadLimit', type: 'number' },
    { name: 'perDayTrfInwardLimit', type: '' },
    { name: 'txnLoadCount', type: 'number' },
    { name: 'txnLTfrInwardCount', type: '' },
    { name: 'perDayUnLoadLimit', type: 'number' },
    { name: 'perDayTfrOutwardLimit', type: '' },
    { name: 'txnUnloadCount', type: 'number' },
    { name: 'txnTrfOutwardCount', type: '' },
    { name: 'perTransaction', type: 'number' }
  ];

  displayedColumns: string[] = this.colDef.map(t => t.name);
  dataSource: MatTableDataSource<any> = new MatTableDataSource();
  dSource: MatTableDataSource<any> = new MatTableDataSource();
  isEditable: boolean = false;
  isEditChecked: boolean = false;

  constructor(
    public dialog: MatDialog,
    private kycService: KycService,
    private cd: ChangeDetectorRef,
    private toastr: ToastrService
  ) {}

  ngOnInit(): void {
    this.kycService.getKycDetails().subscribe(res => {
      console.log("API Response:", res);
      if (res && res.success && Array.isArray(res.data)) {
        const activeRecords = res.data.filter(
          (record: any) => (record.status || '').toLowerCase() === 'active'
        );
        console.log("Filtered Active Records:", activeRecords);
        this.dataSource.data = activeRecords;
        this.dSource.data = JSON.parse(JSON.stringify(activeRecords));
      } else {
        console.warn("No valid data returned.");
      }
    });
  }

  onEdit() {
    this.isEditable = true;
    this.isEditChecked = true;
  }

  onCancel() {
    this.isEditable = false;
    this.isEditChecked = false;
    this.dataSource.data.forEach((x: any) => x.isChecked = false);
    this.dataSource.data = JSON.parse(JSON.stringify(this.dSource.data));
  }

  onSubmit(event: any) {
    const updatedData = this.dataSource.data.filter((x: any) => x.isChecked);
    const payload = updatedData.map(({ entityName, isChecked, ...rest }) => {
      rest.perDayTrfInwardLimit = rest.perDayLoadLimit;
      rest.txnLTfrInwardCount = rest.txnLoadCount;
      rest.perDayTfrOutwardLimit = rest.perDayUnLoadLimit;
      rest.txnTrfOutwardCount = rest.txnUnloadCount;
      return rest;
    });

    if (payload.length > 0) {
      this.kycService.updateKyc(payload).subscribe(
        data => {
          if (data && data.success) {
            this.toastr.success('Update successful');
            this.isEditable = false;
            this.isEditChecked = false;
            this.dataSource.data.forEach((x: any) => x.isChecked = false);
            this.dSource.data = JSON.parse(JSON.stringify(this.dataSource.data));
          } else {
            this.toastr.error(data?.message || 'Update failed');
          }
        },
        error => {
          console.error("Update Error:", error);
          this.toastr.error('Server error');
        }
      );
    } else {
      this.toastr.info('No changes selected');
    }
  }
}






import { ChangeDetectorRef, Component, OnInit } from '@angular/core';
import { MatCheckboxChange } from '@angular/material/checkbox';
import { MatDialog } from '@angular/material/dialog';
import { MatTableDataSource } from '@angular/material/table';
import { ToastrService } from 'ngx-toastr';
import { KycService } from 'src/app/Services/kyc-management.service';

@Component({
  selector: 'app-kyc-management',
  templateUrl: './kyc-management.component.html',
  styleUrls: ['./kyc-management.component.scss']
})
export class KycManagementComponent implements OnInit {
  colDef: any[] = [
    { name: 'type', type: 'readable', readonly: true, heading: '', colspan: 1, subHeading: '' },
    { name: 'capacityLimit', type: 'number', heading: '', colspan: 1, subHeading: '' },
    { name: 'perDayLoadLimit', type: 'number', heading: 'Per day Load', colspan: 2, subHeading: '(Amount)' },
    { name: 'perDayTrfInwardLimit', type: '', heading: 'Per day Transfer', colspan: 0, subHeading: '(inward Amount)' },
    { name: 'txnLoadCount', type: 'number', heading: 'No. of Load', colspan: 2, subHeading: '(Count)' },
    { name: 'txnLTfrInwardCount', type: '', heading: 'No of transfer', colspan: 0, subHeading: '(inward count)' },
    { name: 'perDayUnLoadLimit', type: 'number', heading: 'Per Day Unload', colspan: 2, subHeading: '(Amount)' },
    { name: 'perDayTfrOutwardLimit', type: '', heading: 'Per day transfer', colspan: 0, subHeading: '(Outward Amount)' },
    { name: 'txnUnloadCount', type: 'number', heading: 'No. of unload', colspan: 2, subHeading: '(Count)' },
    { name: 'txnTrfOutwardCount', type: '', heading: 'No. of transfer', colspan: 0, subHeading: '(Outward Count)' },
    { name: 'perTransaction', type: 'number', heading: 'Per Transaction', colspan: 1, subHeading: '' }
  ];

  displayedColumns: string[] = this.colDef.map(t => t.name);
  dataSource: MatTableDataSource<any>;
  dSource: MatTableDataSource<any>;
  isEditable: boolean = false;
  isEditChecked: boolean = false;

  constructor(
    public dialog: MatDialog,
    private kycService: KycService,
    private cd: ChangeDetectorRef,
    private toastr: ToastrService
  ) {}

  ngOnInit(): void {
    this.fetchActiveRecords();
  }

  fetchActiveRecords(): void {
    this.kycService.getKycDetails().subscribe(res => {
      console.log("API Response:", res);
      if (res && res.success) {
        const activeRecords = res.data.filter((record: any) =>
          typeof record.status === 'string' && record.status.toLowerCase() === 'active'
        );
        console.log("Active Records:", activeRecords);
        this.dataSource = new MatTableDataSource(activeRecords);
        this.dSource = new MatTableDataSource(JSON.parse(JSON.stringify(activeRecords)));
        this.cd.detectChanges();
      }
    });
  }

  onEdit() {
    this.isEditable = true;
    this.isEditChecked = true;
  }

  showOptions(event: MatCheckboxChange): void {
    this.displayedColumns = this.colDef.map(t => t.name);
  }

  onCancel() {
    this.isEditable = false;
    this.isEditChecked = false;
    this.dataSource?.filteredData?.forEach((x: any) => x.isChecked = false);
    this.dataSource.data = JSON.parse(JSON.stringify(this.dSource.data));
  }

  onInputChange(value: number, element: any, def: any) {
    if (value < 0) {
      element[def.name] = -value;
    }
  }

  getStatus(element: any): string {
    const capacityLimit = element.capacityLimit;
    if (capacityLimit === 1000000) return 'Normal';
    if (capacityLimit === 5000 || capacityLimit === true) return 'Cooling';
    if (capacityLimit === 15000) return 'IOSCooling';
    if (capacityLimit === false) return 'Normal';
    return 'Normal';
  }

  onSubmit(event: any) {
    const updatedData = this.dataSource?.filteredData?.filter((x: any) => x.isChecked);

    const payload = updatedData.map(({ entityName, isChecked, ...rest }) => {
      rest.perDayTrfInwardLimit = rest.perDayLoadLimit;
      rest.txnLTfrInwardCount = rest.txnLoadCount;
      rest.perDayTfrOutwardLimit = rest.perDayUnLoadLimit;
      rest.txnTrfOutwardCount = rest.txnUnloadCount;
      return rest;
    });

    if (payload.length > 0) {
      this.kycService.updateKyc(payload).subscribe(data => {
        if (data && data.success) {
          this.toastr.success('Update successful');
          this.isEditable = false;
          this.isEditChecked = false;
          this.fetchActiveRecords(); // Refresh table
        } else if (data && data.message) {
          this.toastr.error(data.message);
        } else {
          this.toastr.error('Server error');
        }
      });
    }
  }
}



-- Assumptions:
-- - The working table is rtsp.token_tranlog
-- - Input variables:
--     :SPONSOR_WALLET_ID (payer_wallet)
--     :TXN_COUNT (e.g., 2)
-- - Each transaction has:
--     payer_wallet, payee_wallet, amount, type, status, balance
-- - Only successful transactions are considered (status = 'C')
-- - Fixed disbursement amount = 5000
-- - A virtual table called eligible_wallets is defined globally via a WITH clause

-- Define the eligible_wallets subquery for reuse
WITH eligible_wallets AS (
  SELECT payee_wallet
  FROM rtsp.token_tranlog
  WHERE payer_wallet = :SPONSOR_WALLET_ID
    AND status = 'C'
    AND amount = 5000
  GROUP BY payee_wallet
  HAVING COUNT(*) = :TXN_COUNT
)

-- 1. Count of eligible payee_wallets
SELECT COUNT(*) AS total_eligible_wallets
FROM eligible_wallets;

-- 2. Total value received from sponsor by eligible wallets
SELECT SUM(amount) AS total_value_from_sponsor
FROM rtsp.token_tranlog
WHERE payer_wallet = :SPONSOR_WALLET_ID
  AND status = 'C'
  AND amount = 5000
  AND payee_wallet IN (SELECT payee_wallet FROM eligible_wallets);

-- 3. Total value received by these wallets from any other payer
SELECT SUM(amount) AS total_value_from_others
FROM rtsp.token_tranlog
WHERE status = 'C'
  AND payee_wallet IN (SELECT payee_wallet FROM eligible_wallets)
  AND payer_wallet != :SPONSOR_WALLET_ID;

-- 4. Total value paid (outgoing) by these payee_wallets
SELECT SUM(amount) AS total_value_paid_out
FROM rtsp.token_tranlog
WHERE status = 'C'
  AND payer_wallet IN (SELECT payee_wallet FROM eligible_wallets);

-- 5. Current balance in each of these wallets
SELECT payee_wallet, MAX(balance) AS current_balance
FROM rtsp.token_tranlog
WHERE payee_wallet IN (SELECT payee_wallet FROM eligible_wallets)
GROUP BY payee_wallet;

-- 6. Wallet balance range buckets
SELECT
  CASE
    WHEN balance = 0 THEN 'Zero'
    WHEN balance BETWEEN 1 AND 2499 THEN '₹1–₹2499'
    WHEN balance BETWEEN 2500 AND 4999 THEN '₹2500–₹4999'
    WHEN balance BETWEEN 5000 AND 9999 THEN '₹5000–₹9999'
    WHEN balance = 10000 THEN '₹10000 (full)'
    WHEN balance > 10000 THEN '> ₹10000'
    ELSE 'Unknown'
  END AS balance_range,
  COUNT(DISTINCT payee_wallet) AS wallet_count
FROM rtsp.token_tranlog
WHERE payee_wallet IN (SELECT payee_wallet FROM eligible_wallets)
GROUP BY
  CASE
    WHEN balance = 0 THEN 'Zero'
    WHEN balance BETWEEN 1 AND 2499 THEN '₹1–₹2499'
    WHEN balance BETWEEN 2500 AND 4999 THEN '₹2500–₹4999'
    WHEN balance BETWEEN 5000 AND 9999 THEN '₹5000–₹9999'
    WHEN balance = 10000 THEN '₹10000 (full)'
    WHEN balance > 10000 THEN '> ₹10000'
    ELSE 'Unknown'
  END;









WITH eligible_wallets AS (
  SELECT payee_wallet
  FROM rtsp.token_tranlog
  WHERE payer_wallet = 'SPONSOR_WALLET_ID'
    AND amount = 5000
    AND status = 'C'
  GROUP BY payee_wallet
  HAVING COUNT(*) = 2
)
SELECT
  t.payee_wallet,
  SUM(t.amount) AS amount_received_from_others
FROM rtsp.token_tranlog t
JOIN eligible_wallets ew
  ON t.payee_wallet = ew.payee_wallet
WHERE t.status = 'C'
  AND t.payer_wallet != 'SPONSOR_WALLET_ID'
GROUP BY t.payee_wallet;


WITH eligible_wallets AS (
  SELECT payee_wallet
  FROM rtsp.token_tranlog
  WHERE payer_wallet = 'SPONSOR_WALLET_ID'
    AND amount = 5000
    AND status = 'C'
  GROUP BY payee_wallet
  HAVING COUNT(*) = 2
)
SELECT
  t.payee_wallet,
  SUM(t.amount) AS total_load_amount
FROM rtsp.token_tranlog t
JOIN eligible_wallets ew
  ON t.payee_wallet = ew.payee_wallet
WHERE t.status = 'C'
  AND t.type = 'LOAD'
GROUP BY t.payee_wallet;



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













