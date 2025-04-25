onSubmit(event: any) {
  const updatedData = this.dataSource?.filteredData?.filter((x: any) => x.isChecked && x.status === 'active');

  const payload = updatedData.map(({ entityName, isChecked, ...rest }) => {
    // Create the new updated record with status "active"
    const newRecord = {
      ...rest,
      entityName,
      status: 'active', // NEW record is active
      perDayTrfInwardLimit: rest.perDayLoadLimit,
      txnLtfrInwardCount: rest.txnLoadCount,
      perDayTrfOutwardLimit: rest.perDayUnLoadLimit,
      txnTrfOutwardCount: rest.txnUnloadCount
    };

    return {
      oldEntityId: rest.id || rest.entityId, // For marking old as inactive
      newRecord: newRecord
    };
  });

  // Call backend API to deactivate old + insert new record
  this.yourService.updateWithNewRecord(payload).subscribe(response => {
    console.log('Entity updated with new rows:', response);

    // Optionally re-fetch only active data to refresh UI
    this.fetchActiveData();
  });
}
