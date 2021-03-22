# Read-OTP-from-Message
READ OTP FROM MESSAGE

Android Read OTP from Receive message
OtpActivity.java

    import android.app.Activity;
    import android.app.PendingIntent;
    import android.content.Intent;
    import android.content.IntentFilter;
    import android.content.IntentSender;
    import android.os.Bundle;
    import android.os.CountDownTimer;
    import android.text.Editable;
    import android.text.TextWatcher;
    import android.view.View;
    import android.widget.TextView;
    import android.widget.Toast;

    import androidx.annotation.NonNull;
    import androidx.annotation.Nullable;
    import androidx.appcompat.widget.AppCompatButton;
    import androidx.core.content.ContextCompat;

    import com.google.android.gms.auth.api.Auth;
    import com.google.android.gms.auth.api.credentials.Credential;
    import com.google.android.gms.auth.api.credentials.HintRequest;
    import com.google.android.gms.auth.api.phone.SmsRetriever;
    import com.google.android.gms.common.ConnectionResult;
    import com.google.android.gms.common.api.GoogleApiClient;
    import com.travel.drnme.BaseActivity;
    import com.travel.drnme.R;
    import com.travel.drnme.utilities.OtpEditText;
    import com.travel.drnme.utilities.Utils;

    import butterknife.BindView;
    import butterknife.ButterKnife;
    import butterknife.Unbinder;

    public class OtpActivity extends BaseActivity implements GoogleApiClient.ConnectionCallbacks,
        OtpReceivedInterface, GoogleApiClient.OnConnectionFailedListener {

    public static final long OTP_TIMER_MILLIS = 120 * 1000;
    private final int RESOLVE_HINT = 2;
    GoogleApiClient mGoogleApiClient;
    SmsBroadcastReceiver mSmsBroadcastReceiver;
    @BindView(R.id.tvOtpMsg)
    TextView tvOtpMsg;
    @BindView(R.id.etOTP)
    OtpEditText etOTP;
    @BindView(R.id.tvTimer)
    TextView tvTimer;
    @BindView(R.id.btnVerify)
    AppCompatButton btnVerify;
    @BindView(R.id.tvResendOTP)
    TextView tvResendOTP;
    Unbinder unbinder;
    OTPTimer timer;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_otp);
        unbinder = ButterKnife.bind(this);

        // init broadcast receiver
        mSmsBroadcastReceiver = new SmsBroadcastReceiver();
        //set google api client for hint request
        mGoogleApiClient = new GoogleApiClient.Builder(this)
                .addConnectionCallbacks(this)
                .enableAutoManage(this, this)
                .addApi(Auth.CREDENTIALS_API)
                .build();
        mSmsBroadcastReceiver.setOnOtpListeners(this);
        IntentFilter intentFilter = new IntentFilter();
        intentFilter.addAction(SmsRetriever.SMS_RETRIEVED_ACTION);
        getApplicationContext().registerReceiver(mSmsBroadcastReceiver, intentFilter);
        // get mobile number from phone
        getHintPhoneNumber();

        etOTP.addTextChangedListener(new TextWatcher() {
            @Override
            public void beforeTextChanged(CharSequence s, int start, int count, int after) {
            }

            @Override
            public void onTextChanged(CharSequence s, int start, int before, int count) {
                btnVerify.setEnabled(s.length() == 4);
            }

            @Override
            public void afterTextChanged(Editable s) {
            }
        });
        startTimer();
    }

    private void startTimer() {
        stopTimer();
        timer = new OTPTimer();
        timer.start();
    }

    private void stopTimer() {
        if (timer != null) {
            timer.cancel();
            timer = null;
        }
    }

    @Override
    public void onConnected(@Nullable Bundle bundle) {
    }

    @Override
    public void onConnectionSuspended(int i) {
    }

    @Override
    public void onOtpReceived(String otp) {
        Toast.makeText(this, "Otp Received " + otp, Toast.LENGTH_LONG).show();
        etOTP.setText(otp);
        btnVerify.setEnabled(true);
    }

    @Override
    public void onOtpTimeout() {
        Toast.makeText(this, "Time out, please resend", Toast.LENGTH_LONG).show();
    }

    @Override
    public void onConnectionFailed(@NonNull ConnectionResult connectionResult) {
    }

    public void getHintPhoneNumber() {
        HintRequest hintRequest =
                new HintRequest.Builder()
                        .setPhoneNumberIdentifierSupported(true)
                        .build();
        PendingIntent mIntent = Auth.CredentialsApi.getHintPickerIntent(mGoogleApiClient, hintRequest);
        try {
            startIntentSenderForResult(mIntent.getIntentSender(), RESOLVE_HINT, null, 0, 0, 0);
        } catch (IntentSender.SendIntentException e) {
            e.printStackTrace();
        }
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, @Nullable Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        //Result if we want hint number
        if (requestCode == RESOLVE_HINT) {
            if (resultCode == Activity.RESULT_OK) {
                if (data != null) {
                    Credential credential = data.getParcelableExtra(Credential.EXTRA_KEY);
                    // credential.getId();  < – will need to process phone number string

                }
            }
        }
    }


    @Override
    protected void onDestroy() {
        super.onDestroy();
        unbinder.unbind();
        stopTimer();
    }

    private class OTPTimer extends CountDownTimer {

        OTPTimer() {
            super(OTP_TIMER_MILLIS, 1000);
            tvTimer.setVisibility(View.VISIBLE);
        
            btnVerify.setEnabled(false);
        }

        @Override
        public void onTick(long l) {
            tvTimer.setText(Utils.millisecondsToMinutesAndSeconds(l));
        }

        @Override
        public void onFinish() {
            tvTimer.setVisibility(View.GONE);
            btnVerify.setEnabled(false)

        }
    }
    }
    
    ![Screenshot 2021-03-22 at 7 06 04 PM](https://user-images.githubusercontent.com/12294662/111998128-b8882f80-8b41-11eb-8bf3-197531a5a0f5.png)

interface OtpReceivedInterface 

        public interface OtpReceivedInterface {
            void onOtpReceived(String otp);
            void onOtpTimeout();
        }
SmsBroadcastReceiver.java

    import android.content.BroadcastReceiver;
    import android.content.Context;
    import android.content.Intent;
    import android.os.Bundle;
    import android.util.Log;

    import com.google.android.gms.auth.api.phone.SmsRetriever;
    import com.google.android.gms.common.api.Status;
    import com.google.android.gms.common.api.CommonStatusCodes;

    public class SmsBroadcastReceiver extends BroadcastReceiver {
        private static final String TAG = "SmsBroadcastReceiver";
        OtpReceivedInterface otpReceiveInterface = null;
        public void setOnOtpListeners(OtpReceivedInterface otpReceiveInterface) {
            this.otpReceiveInterface = otpReceiveInterface;
        }
        @Override public void onReceive(Context context, Intent intent) {
            Log.d(TAG, "onReceive: ");
            if (SmsRetriever.SMS_RETRIEVED_ACTION.equals(intent.getAction())) {
                Bundle extras = intent.getExtras();
                Status mStatus = (Status) extras.get(SmsRetriever.EXTRA_STATUS);
                switch (mStatus.getStatusCode()) {
                    case CommonStatusCodes.SUCCESS:
                        // Get SMS message contents'
                        String message = (String) extras.get(SmsRetriever.EXTRA_SMS_MESSAGE);
                        Log.d(TAG, "onReceive: failure "+message);
                        if (otpReceiveInterface != null) {
                            String otp = message.replace("<#> Your otp code is : ", "");
                            otpReceiveInterface.onOtpReceived(otp);
                        }
                        break;
                    case CommonStatusCodes.TIMEOUT:
                        // Waiting for SMS timed out (5 minutes)
                        Log.d(TAG, "onReceive: failure");
                        if (otpReceiveInterface != null) {
                            otpReceiveInterface.onOtpTimeout();
                        }
                        break;
                }
            }
        }
    }




