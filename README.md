# Expansion Files Downloader

Library with Google Packages to download Expansion Files.
Add libaries as submodule in settings.gradle
	include ':app',
	include ':zip_file'
	include ':market_licensing'
	include ':downloader_library'

	project(':zip_file').projectDir = new File('expasionfilesdownloader/zip_file')
	project(':market_licensing').projectDir = new File('expasionfilesdownloader/market_licensing')
	project(':downloader_library').projectDir = new File('expasionfilesdownloader/downloader_library')
	
Make use of the ExpansionFileDownloadActivity in your MainActity.

	 override fun onResume() {
	        println("onResume MainActivity")

	        initExpansionFilesDelivered()

	        if (expansionFileDelivered or !askPermissionAgainPermitted) {
	            startEMCAnimation()
	        } else if (hasNoInternetConnection) {
	            initNoInternetPopup()
	        } else if (downloadIsPaused) {
	            askCellularData()
	        } else {
	            handleDownloadExpansionFiles()
	        }

	        super.onResume()
	    }
	    
    /**
     * Refresh the activity.
     * Show EMC Popup or start downloading expansion files.
     */
    private fun refresh() {
        when {
            expansionFileDelivered or !askPermissionAgainPermitted -> startEMCAnimation()

            downloadIsPaused -> {
                requestDownloadStatus()
                askCellularData()
            }

            isDownloading -> {
                requestDownloadStatus()
                initProgressDialog()
            }

            else -> handleDownloadExpansionFiles()
        }
    }


// Create AlarmReceiver

	package nl.zorgkluis.expansionfiles

	import android.content.BroadcastReceiver
	import android.content.Context
	import android.content.Intent
	import android.content.pm.PackageManager
	import com.google.android.vending.expansion.downloader.DownloaderClientMarshaller

	/**
	 * Created by GeertBerkers.
	 */
	class AppAlarmReceiver : BroadcastReceiver() {

	    override fun onReceive(context: Context, intent: Intent) {
	        try {
	            DownloaderClientMarshaller.startDownloadServiceIfRequired(
	                    context,
	                    intent,
	                    AppDownloaderService::class.java
	            )
	        } catch (e: PackageManager.NameNotFoundException) {
	            e.printStackTrace()
	        }
	    }
	}

// Create Downloader Service

	package nl.zorgkluis.expansionfiles

	import com.google.android.vending.expansion.downloader.impl.DownloaderService

	/**
	 * Created by GeertBerkers.
	 */
	class AppDownloaderService : DownloaderService() {

	    companion object {

	        const val BASE64_PUBLIC_KEY = "..."

	        // TODO: Check if from app, or randomly generated
	        private val SALT = byteArrayOf(1,2,3,4,5,6,7,8,9,10,20)
	    }

	    override fun getPublicKey(): String = BASE64_PUBLIC_KEY

	    override fun getSALT(): ByteArray = SALT

	    override fun getAlarmReceiverClassName(): String = AppAlarmReceiver::class.java.name
	}


// Create MP4ContentProvider

	package nl.zorgkluis.expansionfiles

	import android.net.Uri

	import com.android.vending.expansion.zipfile.APEZProvider

	import java.io.File

	/**
	 * Created by GeertBerkers.
	 */
	class MP4ContentProvider : APEZProvider() {

	    override fun getAuthority(): String {
	        return AUTHORITY
	    }

	    companion object {

	        private const val AUTHORITY = "nl.zorgkluis.expansilfiles.content.MP4ContentProvider"

	        fun buildURI(path: String): Uri {
	            val contentPath = "content://" +
	                    AUTHORITY +
	                    File.separator +
	                    path

	            return Uri.parse(contentPath)
	        }
	    }
	}


// Create xAPK Files
	package nl.zorgkluis.expansionfiles

	/**
	 * Created by GeertBerkers.
	 */
	internal class XAPKFile constructor(
	        val mIsMain: Boolean,
	        val mFileVersion: Int,
	        val mFileSize: Long
	)

// Create ExpansionFilesDownlaoder Activity

@file:Suppress("DEPRECATION")

	package nl.zorgkluis.expansionfiles

	import android.Manifest
	import android.app.PendingIntent
	import android.app.ProgressDialog
	import android.content.ComponentName
	import android.content.DialogInterface
	import android.content.Intent
	import android.content.pm.PackageManager
	import android.os.Bundle
	import android.os.Messenger
	import android.provider.Settings
	import android.support.v4.app.ActivityCompat
	import android.support.v4.content.ContextCompat
	import android.support.v7.app.AlertDialog
	import android.support.v7.app.AppCompatActivity
	import android.util.Log
	import android.widget.Toast
	import com.google.android.vending.expansion.downloader.*
	import nl.zorgkluis.expansionfiles.R
	import org.jetbrains.anko.alert
	import org.jetbrains.anko.appcompat.v7.Appcompat
	import org.jetbrains.anko.progressDialog

	open class ExpansionFileDownloadActivity : AppCompatActivity(), IDownloaderClient {

	    companion object {
	        const val STORAGE_REQUEST = 1
	    }

	    private var cellularDialog: AlertDialog? = null
	    private var noInternetDialog: AlertDialog? = null
	    private var dialogWithProgress: ProgressDialog? = null


	    private var mState: Int = 0
	    private var storagePermissionGranted = false
	    protected var askPermissionAgainPermitted = true
	    protected var expansionFileDelivered: Boolean = false

	    private var mDownloaderClientStub : IStub? = null
	    private var mRemoteService : IDownloaderService? = null

	    private val initialAPKVersionCode : Int = 29
	    private val mainFileSize : Long = 156085389L

	    protected val hasNoInternetConnection
	        get() = mState == IDownloaderClient.STATE_PAUSED_NETWORK_UNAVAILABLE

	    protected val isDownloading
	        get() = dialogWithProgress != null && dialogWithProgress?.progress != dialogWithProgress?.max

	    protected val downloadIsPaused
	        get() = mState == IDownloaderClient.STATE_PAUSED_WIFI_DISABLED_NEED_CELLULAR_PERMISSION ||
	                mState == IDownloaderClient.STATE_PAUSED_NEED_CELLULAR_PERMISSION ||
	                mState == IDownloaderClient.STATE_PAUSED_BY_REQUEST

	    private val xAPKS = arrayOf(
	            XAPKFile(true, initialAPKVersionCode, mainFileSize)
	    )

	    private val maxFileSize : Int by lazy {
	        (xAPKS[0].mFileSize / 1024 / 1024).toInt()
	    }

	    override fun onCreate(savedInstanceState: Bundle?) {
	        super.onCreate(savedInstanceState)
	        createDownloaderStub()
	    }

	    override fun onStart() {
	        connectDownloaderClientStub()
	        super.onStart()
	    }

	    override fun onPause() {
	        hideProgressDialog()
	        super.onPause()
	    }

	    override fun onStop() {
	        disconnectDownloaderClientStub()
	        super.onStop()
	    }
	    /**
	     * Create the sub for our service
	     */
	    private fun createDownloaderStub() {
	        mDownloaderClientStub = DownloaderClientMarshaller.CreateStub(this, AppDownloaderService::class.java)
	    }

	    /**
	     * Connect the stub to our service on start.
	     */
	    private fun connectDownloaderClientStub() {
	        mDownloaderClientStub?.connect(this)
	    }

	    /**
	     * Disconnect the stub from our service on stop
	     */
	    private fun disconnectDownloaderClientStub() {
	        mDownloaderClientStub?.disconnect(this)
	    }

	    /**
	     * Check whether the ExpansionFiles are delivered or not
	     */
	    protected fun initExpansionFilesDelivered() {
	        expansionFileDelivered = false

	        for (xf in xAPKS) {
	            val fileName = Helpers.getExpansionAPKFileName(this, xf.mIsMain, xf.mFileVersion)
	            if (Helpers.doesFileExist(this, fileName, xf.mFileSize, false)) {
	                expansionFileDelivered = true
	            }
	        }
	    }

	    /**
	     * Download expansion files. Ask permission if needed
	     */
	    protected fun handleDownloadExpansionFiles() {
	        if (storagePermissionGranted) {
	            downloadExpansionFiles()
	        } else if (askPermissionAgainPermitted) {
	            handleStoragePermission()
	        }
	    }

	    //region Storage Permission Functions
	    private fun handleStoragePermission() {
	        if (!checkWriteExternalStorage()) {
	            if (shouldShowRequestPermission()) {
	                showNoStoragePermission()
	            } else {
	                requestWriteExternalStoragePermission()
	            }
	            return
	        }

	        storagePermissionGranted = true
	        downloadExpansionFiles()
	    }

	    private fun showNoStoragePermission() {
	        initAppLocale()

	        alert(Appcompat, R.string.askStorageRightsTitle, R.string.askStorageRightsMessage){
	            iconResource = R.drawable.ic_storage

	            positiveButton(R.string.askStorageRightsGrant){
	                requestWriteExternalStoragePermission()
	            }

	            negativeButton(R.string.askStorageRightsCancel){ dialog ->
	                startEMCAnimation()
	                dialog.dismiss()
	            }
	        }.show()
	    }

	    /**
	     * Check permissions for WRITE_EXTERNAL_STORAGE
	     *
	     * @return true if permission granted, false if not
	     */
	    private fun checkWriteExternalStorage(): Boolean {
	        return ContextCompat.checkSelfPermission(this, Manifest.permission.WRITE_EXTERNAL_STORAGE) == PackageManager.PERMISSION_GRANTED
	    }

	    /**
	     * Check if you should show request permission rationale for Write External Storage
	     *
	     * @return true if you should have to show why this permission is needed
	     */
	    private fun shouldShowRequestPermission(): Boolean {
	        return ActivityCompat.shouldShowRequestPermissionRationale(this, Manifest.permission.WRITE_EXTERNAL_STORAGE)
	    }

	    /**
	     * Request permission for Write External Storage
	     */
	    private fun requestWriteExternalStoragePermission() {
	        ActivityCompat.requestPermissions(this, arrayOf(Manifest.permission.WRITE_EXTERNAL_STORAGE), STORAGE_REQUEST)
	    }

	    /**
	     * Check if Never ask again for Write External Storage Permission is pressed!
	     */
	    private fun checkNeverAskPermissionAgain() {
	        if (!shouldShowRequestPermission()) {
	            askPermissionAgainPermitted = checkWriteExternalStorage()
	        }
	    }

	    override fun onRequestPermissionsResult(requestCode: Int, permissions: Array<String>, grantResults: IntArray) {
	        storagePermissionGranted = grantResults.isNotEmpty() && grantResults[0] == PackageManager.PERMISSION_GRANTED

	        if (requestCode == STORAGE_REQUEST) {
	            if (storagePermissionGranted) {
	                downloadExpansionFiles()
	            } else {
	                checkNeverAskPermissionAgain()
	            }
	        }
	    }

	    //endregion

	    /**
	     * Download the expansion files.
	     * Create intent for notification.
	     * Start the download and initialize progressDialog
	     */
	    private fun downloadExpansionFiles() {
	        expansionFileDelivered = false

	        try {
	            val launchIntent = this.intent
	            val mainIntentNotification = Intent(this, this.javaClass).apply {
	                flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TOP
	            }.apply {
	                action = launchIntent.action
	            }

	            if (launchIntent.categories != null) {
	                for (category in launchIntent.categories) {
	                    mainIntentNotification.addCategory(category)
	                }
	            }

	            // Build PendingIntent to open this activity from Notification and request to start the download
	            val pendingIntent = PendingIntent.getActivity(this, 0, mainIntentNotification, PendingIntent.FLAG_UPDATE_CURRENT)
	            val startResult = DownloaderClientMarshaller.startDownloadServiceIfRequired(this, pendingIntent, AppDownloaderService::class.java)

	            if (startResult != DownloaderClientMarshaller.NO_DOWNLOAD_REQUIRED) {
	                initProgressDialog()
	            }
	        } catch (e: PackageManager.NameNotFoundException) {
	            Log.e("Zorgkluis", "Cannot find package!", e)
	        }

	    }

	    protected fun requestDownloadStatus() {
	        mRemoteService?.requestDownloadStatus()
	    }

	    protected fun initProgressDialog() {
	        initAppLocale()

	        if (dialogWithProgress == null) {
	            dialogWithProgress = progressDialog(message = R.string.availableAfterDownloadMessage) {
	                max = maxFileSize
	                setProgressNumberFormat("%1d MB / %2d MB")
	                setProgressStyle(ProgressDialog.STYLE_HORIZONTAL)
	//                setMessage(getString(R.string.available_after_download))
	                setOnDismissListener {
	                    startEMCAnimation()
	                }
	            }
	        }

	        dialogWithProgress?.run {
	            setTitle(R.string.downloading)
	            setMessage(getString(R.string.availableAfterDownloadMessage))
	            show()
	        }
	    }



	    /**
	     * Critical implementation detail. In onServiceConnected we create the
	     * remote service and marshaler. This is how we pass the client information
	     * back to the service so the client can be properly notified of changes. We
	     * must do this every time we reconnect to the service.
	     */
	    override fun onServiceConnected(m: Messenger?) {
	        mRemoteService = DownloaderServiceMarshaller.CreateProxy(m).apply {
	            mDownloaderClientStub?.messenger?.also { messenger ->
	                onClientUpdated(messenger)
	            }
	        }
	    }

	    override fun onDownloadStateChanged(newState: Int) {
	        Log.e("Zorgkluis", "onDownloadStateChanged")
	        setState(newState)
	        val indeterminate: Boolean
	        when (newState) {
	            IDownloaderClient.STATE_IDLE ->
	                // STATE_IDLE means the service is listening, so it's
	                // safe to start making calls via mRemoteService.
	                indeterminate = true

	            IDownloaderClient.STATE_CONNECTING, IDownloaderClient.STATE_FETCHING_URL -> {
	                hideCellularDialog()
	                hideNoInternetDialog()
	                indeterminate = true
	            }

	            IDownloaderClient.STATE_DOWNLOADING -> {
	                initProgressDialog()
	                indeterminate = false
	            }

	            IDownloaderClient.STATE_FAILED_CANCELED, IDownloaderClient.STATE_FAILED, IDownloaderClient.STATE_FAILED_FETCHING_URL -> {
	                indeterminate = false
	                dismissProgressDialog()
	                //TODO: Rename toast
	                Toast.makeText(this, "Probleem opgetreden tijdens downloaden", Toast.LENGTH_LONG).show()
	            }

	            IDownloaderClient.STATE_PAUSED_NEED_CELLULAR_PERMISSION, IDownloaderClient.STATE_PAUSED_WIFI_DISABLED_NEED_CELLULAR_PERMISSION -> {
	                indeterminate = false
	                askCellularData()
	            }

	            IDownloaderClient.STATE_FAILED_UNLICENSED -> {
	                indeterminate = false
	                Toast.makeText(this, "Probleem opgetreden tijdens downloaden", Toast.LENGTH_LONG).show()
	                dialogWithProgress?.dismiss()
	            }

	            IDownloaderClient.STATE_PAUSED_NETWORK_UNAVAILABLE -> {
	                initNoInternetPopup()
	                indeterminate = false
	            }

	            IDownloaderClient.STATE_PAUSED_BY_REQUEST -> indeterminate = false

	            IDownloaderClient.STATE_PAUSED_ROAMING, IDownloaderClient.STATE_PAUSED_SDCARD_UNAVAILABLE -> indeterminate = false

	            IDownloaderClient.STATE_COMPLETED -> {
	                dismissProgressDialog()

	                expansionFileDelivered = true
	                return
	            }
	            else -> indeterminate = true
	        }

	        dialogWithProgress?.isIndeterminate = indeterminate
	    }

	    override fun onDownloadProgress(progressInfo: DownloadProgressInfo) {
	        dialogWithProgress?.run {
	            max = (progressInfo.mOverallTotal / 1024 / 1024).toInt()
	            progress = (progressInfo.mOverallProgress / 1024 / 1024).toInt()
	        }
	    }

	    /**
	     * Dismiss the ProgressDialog
	     */
	    private fun hideProgressDialog() {
	        dialogWithProgress?.hide()
	    }

	    /**
	     * Dismiss the ProgressDialog
	     */
	    private fun dismissProgressDialog() {
	        dialogWithProgress?.dismiss()
	    }

	    protected fun askCellularData() {
	        initAppLocale()


	        if (cellularDialog == null){
	            cellularDialog = alert(Appcompat, R.string.askCellularMessage, R.string.warningTitle){
	                positiveButton(R.string.wifiSettings) { dialog ->
	                    startActivity(Intent(Settings.ACTION_WIFI_SETTINGS))
	                    dialog.cancel()
	                }

	                neutralPressed(R.string.hide) { dialog ->
	                    dialog.dismiss()
	                    startEMCAnimation()
	                }

	                negativeButton(R.string.yes) { dialog ->
	                    mRemoteService?.setDownloadFlags(IDownloaderService.FLAGS_DOWNLOAD_OVER_CELLULAR)
	                    mRemoteService?.requestContinueDownload()
	                    dialog.cancel()

	                    dialogWithProgress?.run {
	                        if (!isShowing) {
	                            show()
	                        }
	                    }
	                }
	            }.build().apply {
	                setCancelable(false)
	            }

	        }

	        hideProgressDialog()
	        hideNoInternetDialog()

	        cellularDialog?.run {
	            setTitle(R.string.warningTitle)
	            setMessage(getString(R.string.askCellularMessage))
	            getButton(DialogInterface.BUTTON_NEGATIVE)?.setText(R.string.yes)
	            getButton(DialogInterface.BUTTON_NEUTRAL)?.setText(R.string.hide)
	            getButton(DialogInterface.BUTTON_POSITIVE)?.setText(R.string.wifiSettings)
	            show()
	        }
	    }

	    private fun hideCellularDialog() {
	        cellularDialog?.hide()
	    }

	    protected fun initNoInternetPopup() {
	        initAppLocale()

	        if (noInternetDialog == null) {
	            noInternetDialog = alert(Appcompat,
	                    R.string.noInternetMessage,
	                    R.string.state_paused_network_unavailable) {

	                positiveButton(R.string.wifiSettings) { dialog ->
	                    startActivity(Intent(Settings.ACTION_WIFI_SETTINGS))
	                    dialog.cancel()
	                }

	                neutralPressed(R.string.hide) { dialog ->
	                    dialog.dismiss()
	                    startEMCAnimation()
	                }

	                negativeButton(R.string.dataSettings) { dialog ->
	                    val intent = Intent().apply {
	                        component = ComponentName(
	                                "com.android.settings",
	                                "com.android.settings.Settings\$DataUsageSummaryActivity"
	                        )
	                    }

	                    startActivity(intent)
	                    dialog.cancel()
	                }

	            }.build().apply {
	                setCancelable(false)
	            }
	        }

	        hideProgressDialog()
	        hideCellularDialog()

	        noInternetDialog?.apply {
	            setTitle(R.string.state_paused_network_unavailable)
	            setMessage(getString(R.string.noInternetMessage))
	            getButton(DialogInterface.BUTTON_NEUTRAL)?.setText(R.string.hide)
	            getButton(DialogInterface.BUTTON_NEGATIVE)?.setText(R.string.dataSettings)
	            getButton(DialogInterface.BUTTON_POSITIVE)?.setText(R.string.wifiSettings)
	        }

	        noInternetDialog?.show()

	    }

	    private fun hideNoInternetDialog() {
	        noInternetDialog?.run {
	            if (isShowing){
	                dismiss()
	            }
	        }

	    }

	    /**
	     * Set the current download state
	     *
	     * @param newState the new state
	     */
	    private fun setState(newState: Int) {
	        if (mState != newState) {
	            mState = newState
	            logState(mState)
	        }
	    }

	    /**
	     * Log the current download state
	     */
	    private fun logState(mState: Int) {
	        val state: String
	        when (mState) {
	            1 -> state = "STATE_IDLE"
	            2 -> state = "STATE_FETCHING_URL"
	            3 -> state = "STATE_CONNECTING"
	            4 -> state = "STATE_DOWNLOADING"
	            5 -> state = "STATE_COMPLETED"
	            6 -> state = "STATE_PAUSED_NETWORK_UNAVAILABLE"
	            7 -> state = "STATE_PAUSED_BY_REQUEST"
	            8 -> state = "STATE_PAUSED_WIFI_DISABLED_NEED_CELLULAR_PERMISSION"
	            9 -> state = "STATE_PAUSED_NEED_CELLULAR_PERMISSION"
	            10 -> state = "STATE_PAUSED_WIFI_DISABLED"
	            11 -> state = "STATE_PAUSED_NEED_WIFI"
	            12 -> state = "STATE_PAUSED_ROAMING"
	            13 -> state = "STATE_PAUSED_NETWORK_SETUP_FAILURE"
	            14 -> state = "STATE_PAUSED_SDCARD_UNAVAILABLE"
	            15 -> state = "STATE_FAILED_UNLICENSED"
	            16 -> state = "STATE_FAILED_FETCHING_URL"
	            17 -> state = "STATE_FAILED_SDCARD_FULL"
	            18 -> state = "STATE_FAILED_CANCELED"
	            19 -> state = "STATE_FAILED"
	            else -> state = "DEFAULT"
	        }
	        Log.e("Zorgkluis", state)
	    }


	    open fun startEMCAnimation() {
	        // Methods to override from MainActivity

	    }

	    open fun initAppLocale() {
	        // Methods to override from MainActivity
	    }
	}


// Create String resources
	<resources>
	    <!-- Downloader service -->
	    <string name="yes">Yes</string>
	    <string name="hide">Hide</string>
	    <string name="warningTitle">Warning</string>
	    <string name="downloading">Downloading</string>
	    <string name="wifiSettings">Wifi settings</string>
	    <string name="dataSettings">Data settings</string>
	    <string name="networkUnavailable">Network unavailable</string>
	    <string name="noInternetMessage">Check your internet settings</string>
	    <string name="availableAfterDownloadMessage">Video\'s are available after download</string>
	    <string name="askCellularMessage">Do you want to download the content with your mobile network? This can cause additional costs</string>
	</resources>

