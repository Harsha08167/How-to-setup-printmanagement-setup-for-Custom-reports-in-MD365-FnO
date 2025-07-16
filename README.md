# How-to-setup-printmanagement-setup-for-Custom-reports-in-MD365-FnO

Step 1:- Extend the PrintMgmtDocumentType Base Enum
PrintMgmtDocumentType

<img width="500" height="168" alt="image" src="https://github.com/user-attachments/assets/a6b1082f-a9f2-4a14-9175-8f417bf277b6" />

Step 2:- Associate new print management document type with a default report design

public static class PrintMgmtDocType_Extension
{
    [SubscribesTo(classStr(PrintMgmtDocType), delegateStr(PrintMgmtDocType, getDefaultReportFormatDelegate))]
    public static void getDefaultReportFormat(PrintMgmtDocumentType _docType, EventHandlerResult _result)
    {
        switch (_docType)
        {
            case PrintMgmtDocumentType::TransferOrder:
                _result.result(ssrsReportStr(HSOETransformOrderCnfReport, Design));
                break;
        }
    }
}


Step 3:- Add PrintMgmtNode according to Module, In my case it is in Inventory Management
[ExtensionOf(classStr(PrintMgmtNode_Invent))]
internal final class PrintMgmtNode_Invent_Class_Extension
{
    public List getDocumentTypes()
    {
        List docTypes = next getDocumentTypes();
        docTypes.addEnd(PrintMgmtDocumentType::TransferOrder);
        return docTypes;
    }
}


Step 4 :- Add PrintMgmtReportFormatPublisher_Extension
[ExtensionOf(classStr(PrintMgmtReportFormatPublisher))]
internal final class PrintMgmtReportFormatPublisher_Extension
{
    [SubscribesTo(classstr(PrintMgmtReportFormatPublisher),
                 delegatestr(PrintMgmtReportFormatPublisher, notifyPopulate))]
    public static void notifyPopulate()
    {
        #PrintMgmtSetup

        void addFormat(PrintMgmtDocumentType _type, PrintMgmtReportFormatName _name,
                   PrintMgmtReportFormatCountryRegionId _countryRegionId = #NoCountryRegionId)
        {
            HSOETOAddPrintMgmtReportFormat::addPrintMgmtReportFormat(_type, _name, _name,
               _countryRegionId);
        }
        
        addFormat(PrintMgmtDocumentType::TransferOrder,
                      ssrsReportStr(HSOETransformOrderCnfReport, Design));
    }

}

Step 5:- Create a new addPrintMgmtReportFormat to use for changing report format
/// <summary>/;
/// Create a new addPrintMgmtReportFormat to use for changing report format
/// </summary>
public class HSOETOAddPrintMgmtReportFormat
{
    /// <summary>
    /// Add new report format in print management method
    /// </summary>
    /// <param name = "_type">PrintMgmtDocumentType</param>
    /// <param name = "_name">PrintMgmtReportFormatName</param>
    /// <param name = "_description">PrintMgmtReportFormatDescription</param>
    /// <param name = "_countryRegionId">PrintMgmtReportFormatCountryRegionId</param>
    /// <param name = "_system">PrintMgmtReportFormatSystem</param>
    /// <param name = "_ssrs">PrintMgmtSSRS</param>
    public static void addPrintMgmtReportFormat(
        PrintMgmtDocumentType _type,
        PrintMgmtReportFormatName _name,
        PrintMgmtReportFormatDescription _description,
        PrintMgmtReportFormatCountryRegionId _countryRegionId,
        PrintMgmtReportFormatSystem _system = false,
        PrintMgmtSSRS _ssrs = PrintMgmtSSRS::SSRS)
    {
        PrintMgmtReportFormat printMgmtReportFormat;
 
        select firstonly printMgmtReportFormat
            where printMgmtReportFormat.DocumentType == _type
                && printMgmtReportFormat.Description == _description
                && printMgmtReportFormat.CountryRegionId == _countryRegionId;
 
        if (!printMgmtReportFormat)
        {
            // Add the new format
            printMgmtReportFormat.clear();
            printMgmtReportFormat.DocumentType = _type;
            printMgmtReportFormat.Name = _name;
            printMgmtReportFormat.Description = _description;
            printMgmtReportFormat.CountryRegionId = _countryRegionId;
            printMgmtReportFormat.System = _system;
            printMgmtReportFormat.ssrs = _ssrs;
            printMgmtReportFormat.insert();
        }
    }

}

Step 6:- Now main class that is Controller class of the report,
﻿/// <summary>
/// The <c>HSOETransferOrderConfirmControllerPrintMgmt</c> class is used to run print management for <c>TransferOrder</c> report.
/// </summary>
internal final class HSOETransferOrderConfirmControllerPrintMgmt extends SrsPrintMgmtController implements BatchRetryable
{
    // Data contract object used to pass parameters to the report
    private HSOETOReportContract contract;
    
    // Stores the current transfer order header record
    private InventTransferTable transTab;

    // List of transfer order records to be processed/printed
    private RecordSortedList transList;

    /// <summary>
    /// Initializes the transaction list based on the record in Args object.
    /// </summary>
    private void initTransList()
    {
        if (this.parmArgs().record())
        {
            // Create a list of journal records based on selected record(s)
            transList = FormLetter::createJournalListCopy(this.parmArgs().record());
        }
        else
        {
            // If the object is directly passed
            transList = this.parmArgs().object();
        }

        if (!transList)
        {
            // Error if no records to print
            throw error("@SYS26348");
        }
    }

    /// <summary>
    /// Gets the type of print (Original, Copy, etc.) from Args.
    /// </summary>
    private PrintCopyOriginal getPrintCopyOriginal()
    {
        PrintCopyOriginal printCopyOriginal = this.parmArgs().parmEnum();
        return printCopyOriginal;
    }

    /// <summary>
    /// Sets up the report format and settings for the current print instance (original or copy).
    /// </summary>
    private void setSettingDetail(PrintMgmtDocInstanceType _type, SRSPrintDestinationSettings _defaultSettings)
    {
        PrintMgmtPrintSettingDetail printSettingDetail = new PrintMgmtPrintSettingDetail();

        // Load default report format for Transfer Order
        PrintMgmtReportFormat printMgmtReportFormat = PrintMgmtReportFormat::findSystem(PrintMgmtDocumentType::TransferOrder);

        printSettingDetail.parmReportFormatName(printMgmtReportFormat.Name);
        printSettingDetail.parmSSRS(printMgmtReportFormat.SSRS);
        printSettingDetail.parmType(_type);
        printSettingDetail.parmInstanceName(enum2str(_type));
        printSettingDetail.parmNumberOfCopies(1); // Only one copy to be printed
        printSettingDetail.parmPrintJobSettings(_defaultSettings);

        // For email medium, you can customize or log email recipients here if needed
        if (_defaultSettings.printMediumType() == SRSPrintMediumType::Email)
        {
            str emailTo = _defaultSettings.emailTo();
        }

        // Load the prepared settings to the printMgmtReportRun object
        printMgmtReportRun.loadSettingDetail(printSettingDetail, '');
    }

    /// <summary>
    /// Core method that performs the actual print management logic for each transfer order.
    /// </summary>
    protected void runPrintMgmt()
    {
        this.initTransList(); // Load the records to be printed
        LanguageId lId = CustTable::find(InventLocation::find(transTab.InventLocationIdTo).AccountNum).languageId();

        if (!transList.first(transTab))
        {
            throw error("@SYS26348");
        }

        // Loop through each record in the list
        do
        {
            PrintCopyOriginal printCopyOriginal = this.getPrintCopyOriginal();

            if (printCopyOriginal == PrintCopyOriginal::OriginalPrint)
            {
                // Load report as an original for printing
                printMgmtReportRun.load(transTab, transTab, lId);
            }
            else if (printCopyOriginal == PrintCopyOriginal::Copy)
            {
                // Apply print settings for copy
                this.setSettingDetail(PrintMgmtDocInstanceType::Copy, printMgmtReportRun.parmDefaultCopyPrintJobSettings());
            }
            else if (printCopyOriginal == PrintCopyOriginal::Original)
            {
                // Apply print settings for original
                this.setSettingDetail(PrintMgmtDocInstanceType::Original, printMgmtReportRun.parmDefaultOriginalPrintJobSettings());
            }
            else if (printCopyOriginal == PrintCopyOriginal::OriginalPrint)
            {
                // Again applying original settings (duplicate case for safety)
                this.setSettingDetail(PrintMgmtDocInstanceType::Original, printMgmtReportRun.parmDefaultOriginalPrintJobSettings());
            }

            // Execute print if there is anything to print
            if (printMgmtReportRun.more())
            {
                this.outputReports();
            }
            else
            {
                warning("@SYS78951");
            }
        }
        while (transList.next(transTab)); // Continue for next transfer order
    }

    /// <summary>
    /// Initializes the PrintMgmtReportRun instance with proper document type and hierarchy.
    /// </summary>
    protected void initPrintMgmtReportRun()
    {
        printMgmtReportRun = PrintMgmtReportRun::construct(PrintMgmtHierarchyType::Invent, PrintMgmtNodeType::Invent, PrintMgmtDocumentType::TransferOrder);
        super(); // Call base method to complete setup
    }

    /// <summary>
    /// Entry point of the controller. Sets parameters, validates, and initiates report execution.
    /// </summary>
    public static void main(Args _args)
    {
        InventTransferTable transTab = _args.record();

        HSOETransferOrderConfirmControllerPrintMgmt srsReportRun = new HSOETransferOrderConfirmControllerPrintMgmt();
        srsReportRun.parmReportName(ssrsReportStr(HSOETransformOrderCnfReport, Design));
        srsReportRun.parmArgs(_args);
        srsReportRun.parmShowDialog(false);
        srsReportRun.initArgsData(_args);

        // Set contract details
        HSOETOReportContract contract = srsReportRun.parmReportContract().parmRdpContract() as HSOETOReportContract;
        contract.parmInventTransferId(transTab.TransferId);
        contract.parmDocumentTitle();

        // Set caption on dialog based on print type
        if (_args.parmEnum() == PrintCopyOriginal::Copy)
        {
            srsReportRun.parmDialogCaption(strFmt("Show Copy Confirmation - %1", transTab.TransferId));
        }
        else if (_args.parmEnum() == PrintCopyOriginal::Original)
        {
            srsReportRun.parmDialogCaption(strFmt("Show Original Confirmation - %1", transTab.TransferId));
        }
        else if (_args.parmEnum() == PrintCopyOriginal::OriginalPrint)
        {
            srsReportRun.parmDialogCaption(strFmt("Show Confirmation - %1", transTab.TransferId));
        }

        // Validation before running the operation
        if (transTab.InventLocationIdTo)
        {
            srsReportRun.validateBeforeStartOps(transTab.TransferId);
        }

        // Execute report
        srsReportRun.startOperation();
    }

    /// <summary>
    /// Extracts and validates the record from Args, sets it as class member.
    /// </summary>
    protected void initArgsData(Args _args)
    {
        if (_args.dataset() != tableNum(InventTransferTable))
        {
            throw error(Error::wrongUseOfFunction(funcName()));
        }

        transTab = InventTransferTable::findRecId(_args.record().RecId);
    }

    /// <summary>
    /// Modifies report contract just before report execution. Sets title, language, and header record.
    /// </summary>
    protected void preRunModifyContract()
    {
        LanguageId languageId = CustTable::find(InventLocation::find(transTab.InventLocationIdTo).AccountNum).languageId();

        if (!contract && this.parmReportContract().parmRdpContract() is HSOETOReportContract)
        {
            contract = this.parmReportContract().parmRdpContract();
        }

        if (contract is HSOETOReportContract)
        {
            contract.parmFormLetterRecordId(transTab.RecId);

            if (this.getPrintCopyOriginal() == PrintCopyOriginal::Copy)
            {
                contract.parmDocumentTitle("Transfer order copy");
            }
            else
            {
                contract.parmDocumentTitle("Transfer order");
            }
        }

        // Set language for report labels and content
        this.parmReportContract().parmRdlContract().parmLanguageId(languageId);
        this.parmReportContract().parmRdlContract().parmLabelLanguageId(languageId);

        super();
    }

    /// <summary>
    /// Validates required fields before starting operation. Returns false if validation fails.
    /// </summary>
    public boolean validateBeforeStartOps(InventTransferId _transferId)
    {
        InventTransferLine transferLine;
        boolean ret = true;

        while select transferLine
            where transferLine.TransferId == _transferId
        {
            // Run custom validations on each transfer line
            ret = transferLine.hsoeValidateFields() && ret;
        }

        // Run custom helper class validation
        ret = ret && HSOEVariantRestrictionsHelper::validateTransferOrder(_transferId);

        if (!ret)
        {
            throw error('Operation failed');
        }

        return ret;
    }

    /// <summary>
    /// Specifies whether this batch job is retryable or not.
    /// </summary>
    [Hookable(false)]
    final boolean isRetryable()
    {
        return true;
    }
}

To Print it via email or standard customer purpose use this class(Extension)

﻿/// <summary>
/// Extension of the <c>PrintMgmtDelegatesHandler</c> class to override party type logic for print management.
/// This is used to specify the party (e.g., Customer/Vendor/None) for a specific document type.
/// </summary>
[ExtensionOf(classStr(PrintMgmtDelegatesHandler))]
internal final class HSOE_PrintMgmtDelegatesHandler_Class_Extension
{
    /// <summary>
    /// Overrides the default party type used in Print Management for the Transfer Order document.
    /// </summary>
    /// <param name="_docType">The document type being printed.</param>
    /// <param name="_jour">The source journal/record being printed.</param>
    /// <returns>
    /// The party type (e.g., Customer, Vendor) to which the document is directed.
    /// </returns>
    protected static PrintMgmtPrintDestinationPartyType getPartyType(PrintMgmtDocumentType _docType, Common _jour)
    {
        // Call the base implementation to get the default party type
        PrintMgmtPrintDestinationPartyType partyType = next getPartyType(_docType, _jour);

        // For Transfer Orders, override the party type to 'Customer'
        if (_docType == PrintMgmtDocumentType::TransferOrder)
        {
            partyType = PrintMgmtPrintDestinationPartyType::Customer;
        }

        return partyType;
    }
}

Ensure that the 'System' field is enabled 
<img width="837" height="446" alt="image" src="https://github.com/user-attachments/assets/f968b622-1a80-4d12-91b3-90e2bde1f3ae" />

If not then use this Runnable class
﻿/// <summary>
/// The <c>HSOEPrintMgmtInitializer</c> class is used to update the Print Management Report Format configuration.
/// Specifically, it sets the selected report format as a "system" format for the Transfer Order document.
/// This ensures that the format becomes available and active for print management usage in the system.
/// </summary>
internal final class HSOEPrintMgmtInitializer
{
    /// <summary>
    /// Entry point for this class. Can be called manually from a menu item.
    /// This method checks if the report format for Transfer Order is already marked as system,
    /// and if not, updates the flag to mark it as system-defined.
    /// </summary>
    /// <param name = "_args">The arguments passed by the system or menu item (not used here).</param>
    public static void main(Args _args)
    {
        PrintMgmtReportFormat format;

        // Start a transaction to ensure data consistency
        ttsBegin;

        // Query the report format for Transfer Order that matches the specific SSRS report design
        select forUpdate format
            where format.Name == 'HSOEtransformOrderCnfReport.Design' // Name must exactly match the format configured in PrintMgmt
               && format.DocumentType == PrintMgmtDocumentType::TransferOrder;

        // If the format exists and is not already marked as a system format, update it
        if (format && !format.System)
        {
            format.System = NoYes::Yes;  // Mark the report format as a system-defined format
            format.doUpdate();           // Save changes to the database
            info(strFmt("System flag enabled for format %1", format.Name));
        }

        // Commit the transaction
        ttsCommit;
    }
}

Add controller class in 3 Output menu item buttons
i.e, Copy, Original and Original Print

Note:- this code will work all print management conditions, Email, Printer etc., 


