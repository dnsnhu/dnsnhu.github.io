---
layout: default
title: "Enhanced Source"
description: "All 53 Java classes of the weight tracking application after the three capstone enhancements."
permalink: /artifact/enhanced/
---


# Enhanced Source

The application after all three enhancements. 53 classes grouped by package. Expand any file to read it here without downloading anything.

[Back to the portfolio]({{ site.baseurl }}/)

## Contents

- [User interface](#user-interface) (5 files)
- [ViewModels](#viewmodels) (5 files)
- [Analytics](#analytics) (7 files)
- [Data layer](#data-layer) (5 files)
- [Entities](#entities) (3 files)
- [Data access](#data-access) (6 files)
- [Models](#models) (4 files)
- [Migration](#migration) (3 files)
- [Security](#security) (2 files)
- [Utilities](#utilities) (5 files)
- [Tests](#tests) (8 files)

## User interface

<details markdown="1">
<summary><code>LoginActivity.java</code></summary>

```java
package com.example.cs360project.ui;

import android.content.Intent;
import android.os.Bundle;
import android.widget.Button;
import android.widget.EditText;
import android.widget.Toast;

import androidx.appcompat.app.AppCompatActivity;
import androidx.lifecycle.ViewModelProvider;

import com.example.cs360project.R;
import com.example.cs360project.security.PasswordHasher;
import com.example.cs360project.viewmodel.LoginViewModel;
import com.example.cs360project.viewmodel.ViewModelFactory;

/**
 * Sign in and registration screen.
 *
 * Reads two fields, hands them to the ViewModel, and renders what comes back.
 *
 * The password is copied out as a character array so it can be cleared once hashed. A
 * String cannot be cleared.
 */
public class LoginActivity extends AppCompatActivity {

    private LoginViewModel viewModel;
    private EditText editUser;
    private EditText editPass;
    private Button btnLogin;
    private Button btnRegister;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_login);

        editUser = findViewById(R.id.editUsername);
        editPass = findViewById(R.id.editPassword);
        btnLogin = findViewById(R.id.buttonLogin);
        btnRegister = findViewById(R.id.buttonRegister);

        viewModel = new ViewModelProvider(this, new ViewModelFactory(this))
                .get(LoginViewModel.class);

        btnLogin.setOnClickListener(v ->
                viewModel.login(editUser.getText().toString(), readPassword()));

        btnRegister.setOnClickListener(v ->
                viewModel.register(editUser.getText().toString(), readPassword()));

        observeViewModel();
    }

    /** Wires the screen to the state the ViewModel publishes. */
    private void observeViewModel() {
        viewModel.getMessage().observe(this, text -> {
            if (text != null) {
                Toast.makeText(this, text, Toast.LENGTH_SHORT).show();
                viewModel.onMessageHandled();
            }
        });

        viewModel.getBusy().observe(this, busy -> {
            boolean enabled = busy == null || !busy;
            btnLogin.setEnabled(enabled);
            btnRegister.setEnabled(enabled);
        });

        viewModel.getSignedIn().observe(this, signedIn -> {
            if (Boolean.TRUE.equals(signedIn)) {
                viewModel.onNavigationHandled();
                editPass.setText("");
                startActivity(new Intent(this, WeightListActivity.class));
                finish();
            }
        });
    }

    /**
     * Copies the password field into an array the ViewModel can clear.
     *
     * @return the entered characters, never null
     */
    private char[] readPassword() {
        return PasswordHasher.toCharArray(editPass.getText());
    }
}
```

</details>

<details markdown="1">
<summary><code>WeightListActivity.java</code></summary>

```java
package com.example.cs360project.ui;

import android.content.Intent;
import android.os.Bundle;
import android.view.Menu;
import android.view.MenuItem;
import android.widget.Toast;

import androidx.annotation.NonNull;
import androidx.appcompat.app.ActionBar;
import androidx.appcompat.app.AppCompatActivity;
import androidx.lifecycle.ViewModelProvider;
import androidx.recyclerview.widget.LinearLayoutManager;
import androidx.recyclerview.widget.RecyclerView;

import com.example.cs360project.R;
import com.example.cs360project.analytics.AnalyticsSnapshot;
import com.example.cs360project.analytics.StreakSummary;
import com.example.cs360project.data.model.WeightItem;
import com.example.cs360project.security.SessionManager;
import com.example.cs360project.viewmodel.ViewModelFactory;
import com.example.cs360project.viewmodel.WeightListViewModel;

import com.google.android.material.floatingactionbutton.FloatingActionButton;

import java.util.Locale;

/**
 * The main list screen.
 *
 * The derived figures go in the action bar subtitle.
 *
 * Both the list and the figures are observed, so a change made anywhere reaches this
 * screen without it asking.
 */
public class WeightListActivity extends AppCompatActivity {

    private WeightListViewModel viewModel;
    private WeightAdapter adapter;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        if (!SessionManager.getInstance().isActive()) {
            returnToLogin();
            return;
        }

        setContentView(R.layout.activity_weight_list);

        RecyclerView recyclerView = findViewById(R.id.recyclerViewWeights);
        FloatingActionButton fabAdd = findViewById(R.id.fabAddWeight);

        adapter = new WeightAdapter(new WeightAdapter.OnItemClickListener() {
            @Override
            public void onDeleteClick(long id) {
                viewModel.delete(id);
            }

            @Override
            public void onEditClick(WeightItem item) {
                Intent intent = new Intent(WeightListActivity.this, AddWeightActivity.class);
                intent.putExtra(AddWeightActivity.EXTRA_ITEM_ID, item.getId());
                intent.putExtra(AddWeightActivity.EXTRA_ITEM_WEIGHT, item.getWeight());
                startActivity(intent);
            }
        });

        recyclerView.setLayoutManager(new LinearLayoutManager(this));
        recyclerView.setAdapter(adapter);

        viewModel = new ViewModelProvider(this, new ViewModelFactory(this))
                .get(WeightListViewModel.class);

        viewModel.getWeights().observe(this, items -> adapter.setItems(items));
        viewModel.getAnalytics().observe(this, this::renderAnalytics);

        viewModel.getMessage().observe(this, text -> {
            if (text != null) {
                Toast.makeText(this, text, Toast.LENGTH_SHORT).show();
                viewModel.onMessageHandled();
            }
        });

        fabAdd.setOnClickListener(v ->
                startActivity(new Intent(this, AddWeightActivity.class)));
    }

    /**
     * Puts the derived figures in the action bar subtitle.
     *
     * @param snapshot the current figures, may be null before the first load
     */
    private void renderAnalytics(AnalyticsSnapshot snapshot) {
        ActionBar bar = getSupportActionBar();
        if (bar == null) {
            return;
        }
        if (snapshot == null || !snapshot.hasData()) {
            bar.setSubtitle(null);
            return;
        }

        StringBuilder text = new StringBuilder();
        text.append(String.format(Locale.US, "avg %.1f", snapshot.getMovingAverage()));
        text.append(String.format(Locale.US, "  low %.1f  high %.1f",
                snapshot.getWindowMin(), snapshot.getWindowMax()));

        StreakSummary streaks = snapshot.getStreaks();
        if (streaks.getCurrentDirection() == StreakSummary.Direction.DOWN
                && streaks.getCurrentLength() > 1) {
            text.append("  down ").append(streaks.getCurrentLength());
        } else if (streaks.getCurrentDirection() == StreakSummary.Direction.UP
                && streaks.getCurrentLength() > 1) {
            text.append("  up ").append(streaks.getCurrentLength());
        }

        Float entriesToGoal = snapshot.getProjection().getEntriesToGoal();
        if (snapshot.getProjection().isAvailable() && entriesToGoal != null) {
            text.append(String.format(Locale.US, "  goal in ~%d", Math.round(entriesToGoal)));
        }

        bar.setSubtitle(text.toString());
    }

    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        getMenuInflater().inflate(R.menu.weight_menu, menu);
        return true;
    }

    @Override
    public boolean onOptionsItemSelected(@NonNull MenuItem item) {
        if (item.getItemId() == R.id.action_settings) {
            startActivity(new Intent(this, SettingsActivity.class));
            return true;
        }
        return super.onOptionsItemSelected(item);
    }

    /** Sends an unauthenticated user back to the sign in screen. */
    private void returnToLogin() {
        Intent intent = new Intent(this, LoginActivity.class);
        intent.addFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP | Intent.FLAG_ACTIVITY_NEW_TASK);
        startActivity(intent);
        finish();
    }
}
```

</details>

<details markdown="1">
<summary><code>AddWeightActivity.java</code></summary>

```java
package com.example.cs360project.ui;

import android.Manifest;
import android.annotation.SuppressLint;
import android.content.pm.PackageManager;
import android.os.Bundle;
import android.telephony.SmsManager;
import android.widget.Button;
import android.widget.EditText;
import android.widget.Toast;

import androidx.activity.result.ActivityResultLauncher;
import androidx.activity.result.contract.ActivityResultContracts;
import androidx.appcompat.app.AppCompatActivity;
import androidx.core.content.ContextCompat;
import androidx.lifecycle.ViewModelProvider;

import java.util.Locale;

import com.example.cs360project.R;
import com.example.cs360project.viewmodel.AddWeightViewModel;
import com.example.cs360project.viewmodel.ViewModelFactory;

/**
 * Add and edit screen.
 *
 * Reads the fields, asks for the SMS permission, and sends the message. Validation and
 * the decision to notify belong to the ViewModel.
 *
 * Permission requests use the Activity Result API.
 */
public class AddWeightActivity extends AppCompatActivity {

    /** Intent extra carrying the row id when an existing entry is being edited. */
    public static final String EXTRA_ITEM_ID = "ITEM_ID";

    /** Intent extra carrying the current weight when an existing entry is being edited. */
    public static final String EXTRA_ITEM_WEIGHT = "ITEM_WEIGHT";

    /**
     * Destination for the goal notification.
     *
     * The emulator's own number, standing in for a real one. A shipping version reads a
     * number the user has entered and verified, since a fixed destination sends the user's
     * health information to whoever happens to own it.
     */
    private static final String NOTIFICATION_RECIPIENT = "5555215554";

    private AddWeightViewModel viewModel;
    private EditText editWeight;
    private EditText editGoal;
    private Button btnSave;
    private long existingId = -1L;

    private ActivityResultLauncher<String> smsPermissionLauncher;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_add_weight);

        editWeight = findViewById(R.id.editWeightInput);
        editGoal = findViewById(R.id.editGoalInput);
        btnSave = findViewById(R.id.buttonSaveWeight);

        smsPermissionLauncher = registerForActivityResult(
                new ActivityResultContracts.RequestPermission(),
                granted -> {
                    if (Boolean.TRUE.equals(granted)) {
                        sendGoalNotification();
                    } else {
                        Toast.makeText(this,
                                "SMS permission denied. Continuing without notification.",
                                Toast.LENGTH_SHORT).show();
                    }
                    viewModel.onNotificationHandled();
                });

        if (getIntent().hasExtra(EXTRA_ITEM_ID)) {
            existingId = getIntent().getLongExtra(EXTRA_ITEM_ID, -1L);
            // The extra is a float now that the model carries a number rather than text,
            // so the screen formats it for the field instead of passing a String around.
            float existingWeight = getIntent().getFloatExtra(EXTRA_ITEM_WEIGHT, 0f);
            editWeight.setText(String.format(Locale.US, "%.1f", existingWeight));
            btnSave.setText(R.string.update_weight);
        }

        viewModel = new ViewModelProvider(this, new ViewModelFactory(this))
                .get(AddWeightViewModel.class);

        btnSave.setOnClickListener(v -> viewModel.save(
                editWeight.getText().toString(),
                editGoal.getText().toString(),
                existingId));

        observeViewModel();
    }

    /** Wires the screen to the state the ViewModel publishes. */
    private void observeViewModel() {
        viewModel.getMessage().observe(this, text -> {
            if (text != null) {
                Toast.makeText(this, text, Toast.LENGTH_SHORT).show();
                viewModel.onMessageHandled();
            }
        });

        viewModel.getBusy().observe(this, busy -> btnSave.setEnabled(busy == null || !busy));

        viewModel.getNotifyGoalReached().observe(this, notify -> {
            if (Boolean.TRUE.equals(notify)) {
                requestNotification();
            }
        });

        viewModel.getFinished().observe(this, finished -> {
            if (Boolean.TRUE.equals(finished)) {
                finish();
            }
        });
    }

    /**
     * Sends the goal notification, requesting the permission first if it is not held.
     *
     * Reaching this point already means the user opted into SMS in settings. The runtime
     * permission is a separate grant and is requested only now, when there is something to
     * send.
     */
    private void requestNotification() {
        boolean granted = ContextCompat.checkSelfPermission(this, Manifest.permission.SEND_SMS)
                == PackageManager.PERMISSION_GRANTED;

        if (granted) {
            sendGoalNotification();
            viewModel.onNotificationHandled();
        } else {
            smsPermissionLauncher.launch(Manifest.permission.SEND_SMS);
        }
    }

    /** Sends the goal reached message. Failures are reported and do not crash the screen. */
    @SuppressLint("UnlocalizedSms")
    private void sendGoalNotification() {
        try {
            SmsManager smsManager = getSystemService(SmsManager.class);
            smsManager.sendTextMessage(NOTIFICATION_RECIPIENT, null,
                    "Goal reached. Congratulations.", null, null);
            Toast.makeText(this, "Goal notification sent", Toast.LENGTH_SHORT).show();
        } catch (RuntimeException e) {
            Toast.makeText(this, "Could not send the notification", Toast.LENGTH_SHORT).show();
        }
    }
}
```

</details>

<details markdown="1">
<summary><code>SettingsActivity.java</code></summary>

```java
package com.example.cs360project.ui;

import android.Manifest;
import android.content.pm.PackageManager;
import android.os.Bundle;
import android.widget.Button;
import android.widget.CompoundButton;
import android.widget.Switch;
import android.widget.Toast;

import androidx.activity.result.ActivityResultLauncher;
import androidx.activity.result.contract.ActivityResultContracts;
import androidx.appcompat.app.AppCompatActivity;
import androidx.core.content.ContextCompat;
import androidx.lifecycle.ViewModelProvider;

import com.example.cs360project.R;
import com.example.cs360project.viewmodel.SettingsViewModel;
import com.example.cs360project.viewmodel.ViewModelFactory;

/**
 * Settings screen.
 *
 * Delete-all goes through the shared repository, so the list screen sees the deletion
 * through the same LiveData.
 *
 * The switch listener is detached while the state is set programmatically. Leaving it
 * attached makes restoring the stored value fire the listener and write it straight back.
 */
public class SettingsActivity extends AppCompatActivity {

    private SettingsViewModel viewModel;
    private Switch smsSwitch;
    private CompoundButton.OnCheckedChangeListener switchListener;
    private ActivityResultLauncher<String> smsPermissionLauncher;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.fragment_settings);

        smsSwitch = findViewById(R.id.sms_int_switch);
        Button btnDeleteAll = findViewById(R.id.delete_all_data);

        viewModel = new ViewModelProvider(this, new ViewModelFactory(this))
                .get(SettingsViewModel.class);

        smsPermissionLauncher = registerForActivityResult(
                new ActivityResultContracts.RequestPermission(),
                granted -> {
                    boolean allowed = Boolean.TRUE.equals(granted);
                    viewModel.setSmsEnabled(allowed);
                    Toast.makeText(this,
                            allowed ? "SMS notifications enabled"
                                    : "Permission denied. SMS notifications stay off.",
                            Toast.LENGTH_SHORT).show();
                });

        switchListener = (buttonView, isChecked) -> {
            if (!isChecked) {
                viewModel.setSmsEnabled(false);
                return;
            }
            boolean granted = ContextCompat.checkSelfPermission(this, Manifest.permission.SEND_SMS)
                    == PackageManager.PERMISSION_GRANTED;
            if (granted) {
                viewModel.setSmsEnabled(true);
            } else {
                smsPermissionLauncher.launch(Manifest.permission.SEND_SMS);
            }
        };

        viewModel.getSmsEnabled().observe(this, this::renderSwitch);

        viewModel.getMessage().observe(this, text -> {
            if (text != null) {
                Toast.makeText(this, text, Toast.LENGTH_SHORT).show();
                viewModel.onMessageHandled();
            }
        });

        btnDeleteAll.setOnClickListener(v -> viewModel.deleteAllWeights());
    }

    /**
     * Sets the switch without triggering the user driven path.
     *
     * @param enabled the state to display
     */
    private void renderSwitch(Boolean enabled) {
        smsSwitch.setOnCheckedChangeListener(null);
        smsSwitch.setChecked(Boolean.TRUE.equals(enabled));
        smsSwitch.setOnCheckedChangeListener(switchListener);
    }
}
```

</details>

<details markdown="1">
<summary><code>WeightAdapter.java</code></summary>

```java
package com.example.cs360project.ui;

import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.ImageButton;
import android.widget.TextView;

import androidx.annotation.NonNull;
import androidx.recyclerview.widget.DiffUtil;
import androidx.recyclerview.widget.RecyclerView;

import com.example.cs360project.R;
import com.example.cs360project.data.model.WeightItem;

import java.util.ArrayList;
import java.util.List;

/**
 * Binds weight entries to rows.
 *
 * Updates go through {@link DiffUtil}, so only changed rows are dispatched and scroll
 * position survives. The callback compares records by row id and contents by the fields
 * the row draws.
 *
 * The diff is O(n + d squared) and runs on the main thread, which these lists are short
 * enough for.
 */
public class WeightAdapter extends RecyclerView.Adapter<WeightAdapter.WeightViewHolder> {

    private final List<WeightItem> weightList = new ArrayList<>();
    private final OnItemClickListener listener;

    /** Row level events the hosting screen reacts to. */
    public interface OnItemClickListener {

        /**
         * @param id the row id to remove
         */
        void onDeleteClick(long id);

        /**
         * @param item the entry to edit
         */
        void onEditClick(WeightItem item);
    }

    public WeightAdapter(OnItemClickListener listener) {
        this.listener = listener;
    }

    /**
     * Replaces the displayed entries, dispatching only what changed.
     *
     * @param items the new list, may be null
     */
    public void setItems(List<WeightItem> items) {
        List<WeightItem> incoming = items == null ? new ArrayList<>() : new ArrayList<>(items);
        DiffUtil.DiffResult diff = DiffUtil.calculateDiff(
                new WeightDiffCallback(weightList, incoming));

        weightList.clear();
        weightList.addAll(incoming);
        diff.dispatchUpdatesTo(this);
    }

    @NonNull
    @Override
    public WeightViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        View view = LayoutInflater.from(parent.getContext())
                .inflate(R.layout.list_item_weight, parent, false);
        return new WeightViewHolder(view);
    }

    @Override
    public void onBindViewHolder(@NonNull WeightViewHolder holder, int position) {
        WeightItem item = weightList.get(position);
        holder.textDate.setText(item.getFormattedDate());
        holder.textWeight.setText(item.getFormattedWeight());

        holder.btnDelete.setOnClickListener(v -> listener.onDeleteClick(item.getId()));
        holder.itemView.setOnClickListener(v -> listener.onEditClick(item));
    }

    @Override
    public int getItemCount() {
        return weightList.size();
    }

    /** Tells DiffUtil which rows are the same record and which of those changed. */
    private static final class WeightDiffCallback extends DiffUtil.Callback {

        private final List<WeightItem> oldList;
        private final List<WeightItem> newList;

        WeightDiffCallback(List<WeightItem> oldList, List<WeightItem> newList) {
            this.oldList = new ArrayList<>(oldList);
            this.newList = newList;
        }

        @Override
        public int getOldListSize() {
            return oldList.size();
        }

        @Override
        public int getNewListSize() {
            return newList.size();
        }

        @Override
        public boolean areItemsTheSame(int oldItemPosition, int newItemPosition) {
            return oldList.get(oldItemPosition).getId() == newList.get(newItemPosition).getId();
        }

        @Override
        public boolean areContentsTheSame(int oldItemPosition, int newItemPosition) {
            WeightItem oldItem = oldList.get(oldItemPosition);
            WeightItem newItem = newList.get(newItemPosition);
            return Float.compare(oldItem.getWeight(), newItem.getWeight()) == 0
                    && oldItem.getRecordedAt() == newItem.getRecordedAt();
        }
    }

    /** Holds the views for one row. */
    public static class WeightViewHolder extends RecyclerView.ViewHolder {

        final TextView textDate;
        final TextView textWeight;
        final ImageButton btnDelete;

        public WeightViewHolder(@NonNull View itemView) {
            super(itemView);
            textDate = itemView.findViewById(R.id.textViewDate);
            textWeight = itemView.findViewById(R.id.textViewWeight);
            btnDelete = itemView.findViewById(R.id.buttonDelete);
        }
    }
}
```

</details>

## ViewModels

<details markdown="1">
<summary><code>LoginViewModel.java</code></summary>

```java
package com.example.cs360project.viewmodel;

import androidx.lifecycle.LiveData;
import androidx.lifecycle.MutableLiveData;
import androidx.lifecycle.ViewModel;

import com.example.cs360project.data.UserRepository;
import com.example.cs360project.data.model.AuthenticatedUser;
import com.example.cs360project.security.PasswordHasher;
import com.example.cs360project.security.SessionManager;
import com.example.cs360project.util.ValidationResult;
import com.example.cs360project.util.Validator;

/**
 * State and behavior for the sign in screen.
 *
 * The activity holds no credentials, no database handle, and no decision logic. It reads
 * the fields, hands them here, and renders whatever this class publishes.
 *
 * The repository is injected, so a JVM unit test can drive this with a fake. Sign in
 * logic sitting in a click listener next to a concrete helper would need an emulator.
 */
public class LoginViewModel extends ViewModel {

    private final UserRepository userRepository;
    private final SessionManager sessionManager;

    private final MutableLiveData<String> message = new MutableLiveData<>();
    private final MutableLiveData<Boolean> signedIn = new MutableLiveData<>(false);
    private final MutableLiveData<Boolean> busy = new MutableLiveData<>(false);

    public LoginViewModel(UserRepository userRepository, SessionManager sessionManager) {
        this.userRepository = userRepository;
        this.sessionManager = sessionManager;
    }

    /** Messages to show the user. Cleared by {@link #onMessageHandled()}. */
    public LiveData<String> getMessage() {
        return message;
    }

    /** True once a sign in has succeeded. Cleared by {@link #onNavigationHandled()}. */
    public LiveData<Boolean> getSignedIn() {
        return signedIn;
    }

    /** True while a request is in flight, so the screen can disable its buttons. */
    public LiveData<Boolean> getBusy() {
        return busy;
    }

    /**
     * Attempts to sign in.
     *
     * @param usernameText raw text from the username field
     * @param passwordText raw characters from the password field; cleared by this method
     */
    public void login(String usernameText, char[] passwordText) {
        String username = usernameText == null ? "" : usernameText.trim();

        ValidationResult usernameCheck = Validator.validateUsername(username);
        if (!usernameCheck.isValid()) {
            PasswordHasher.wipe(passwordText);
            message.setValue(usernameCheck.getMessage());
            return;
        }

        ValidationResult passwordCheck = Validator.validateSubmittedPassword(passwordText);
        if (!passwordCheck.isValid()) {
            PasswordHasher.wipe(passwordText);
            message.setValue(passwordCheck.getMessage());
            return;
        }

        busy.setValue(true);
        userRepository.login(username, passwordText, outcome -> {
            busy.setValue(false);
            if (outcome.isSuccess()) {
                AuthenticatedUser user = outcome.getValue();
                sessionManager.startSession(user.getId(), user.getUsername());
                signedIn.setValue(true);
            } else {
                message.setValue(outcome.getMessage());
            }
        });
    }

    /**
     * Attempts to create an account.
     *
     * @param usernameText raw text from the username field
     * @param passwordText raw characters from the password field; cleared by this method
     */
    public void register(String usernameText, char[] passwordText) {
        String username = usernameText == null ? "" : usernameText.trim();

        ValidationResult usernameCheck = Validator.validateUsername(username);
        if (!usernameCheck.isValid()) {
            PasswordHasher.wipe(passwordText);
            message.setValue(usernameCheck.getMessage());
            return;
        }

        ValidationResult passwordCheck = Validator.validateNewPassword(passwordText);
        if (!passwordCheck.isValid()) {
            PasswordHasher.wipe(passwordText);
            message.setValue(passwordCheck.getMessage());
            return;
        }

        busy.setValue(true);
        userRepository.register(username, passwordText, outcome -> {
            busy.setValue(false);
            message.setValue(outcome.isSuccess()
                    ? "Account created. Please sign in."
                    : outcome.getMessage());
        });
    }

    /** Called by the view once a message has been displayed. */
    public void onMessageHandled() {
        message.setValue(null);
    }

    /** Called by the view once it has navigated away. */
    public void onNavigationHandled() {
        signedIn.setValue(false);
    }
}
```

</details>

<details markdown="1">
<summary><code>WeightListViewModel.java</code></summary>

```java
package com.example.cs360project.viewmodel;

import androidx.lifecycle.LiveData;
import androidx.lifecycle.MutableLiveData;
import androidx.lifecycle.ViewModel;

import com.example.cs360project.analytics.AnalyticsSnapshot;
import com.example.cs360project.data.WeightRepository;
import com.example.cs360project.data.model.WeightItem;

import java.util.List;

/**
 * State and behavior for the list screen.
 *
 * The list and the figures derived from it are both observed, so neither needs a
 * re-query in {@code onResume} or after a delete.
 */
public class WeightListViewModel extends ViewModel {

    private final WeightRepository weightRepository;
    private final MutableLiveData<String> message = new MutableLiveData<>();

    public WeightListViewModel(WeightRepository weightRepository) {
        this.weightRepository = weightRepository;
        this.weightRepository.refresh();
    }

    /** The entries to display, in recording order. */
    public LiveData<List<WeightItem>> getWeights() {
        return weightRepository.getWeights();
    }

    /** The derived figures: moving average, window extremes, streaks, and projection. */
    public LiveData<AnalyticsSnapshot> getAnalytics() {
        return weightRepository.getAnalytics();
    }

    /** Messages to show the user. Cleared by {@link #onMessageHandled()}. */
    public LiveData<String> getMessage() {
        return message;
    }

    /**
     * Removes one entry.
     *
     * @param id the row to remove
     */
    public void delete(long id) {
        weightRepository.delete(id, outcome -> {
            if (!outcome.isSuccess()) {
                message.setValue(outcome.getMessage());
            }
        });
    }

    /** Called by the view once a message has been displayed. */
    public void onMessageHandled() {
        message.setValue(null);
    }
}
```

</details>

<details markdown="1">
<summary><code>AddWeightViewModel.java</code></summary>

```java
package com.example.cs360project.viewmodel;

import androidx.lifecycle.LiveData;
import androidx.lifecycle.MutableLiveData;
import androidx.lifecycle.ViewModel;

import com.example.cs360project.data.SettingsPreferences;
import com.example.cs360project.data.WeightRepository;
import com.example.cs360project.util.ValidationResult;
import com.example.cs360project.util.Validator;

/**
 * State and behavior for the add and edit screen.
 *
 * Validates through {@link Validator}, which reports a message for a bad entry rather
 * than throwing, and decides whether reaching the goal should raise a notification. The
 * activity keeps the part that needs a Context.
 */
public class AddWeightViewModel extends ViewModel {

    private final WeightRepository weightRepository;
    private final SettingsPreferences settingsPreferences;

    private final MutableLiveData<String> message = new MutableLiveData<>();
    private final MutableLiveData<Boolean> notifyGoalReached = new MutableLiveData<>(false);
    private final MutableLiveData<Boolean> finished = new MutableLiveData<>(false);
    private final MutableLiveData<Boolean> busy = new MutableLiveData<>(false);

    public AddWeightViewModel(WeightRepository weightRepository,
                              SettingsPreferences settingsPreferences) {
        this.weightRepository = weightRepository;
        this.settingsPreferences = settingsPreferences;
    }

    /** Messages to show the user. Cleared by {@link #onMessageHandled()}. */
    public LiveData<String> getMessage() {
        return message;
    }

    /**
     * True when the saved weight has reached the goal and the user has opted into SMS.
     * Cleared by {@link #onNotificationHandled()}.
     */
    public LiveData<Boolean> getNotifyGoalReached() {
        return notifyGoalReached;
    }

    /** True when the screen has finished its work and should close. */
    public LiveData<Boolean> getFinished() {
        return finished;
    }

    /** True while a save is in flight. */
    public LiveData<Boolean> getBusy() {
        return busy;
    }

    /**
     * Validates the form and saves it.
     *
     * @param weightText raw text from the weight field
     * @param goalText   raw text from the optional goal field
     * @param existingId the row being edited, or -1 when adding
     */
    public void save(String weightText, String goalText, long existingId) {
        ValidationResult weightCheck = Validator.validateWeight(weightText);
        if (!weightCheck.isValid()) {
            message.setValue(weightCheck.getMessage());
            return;
        }

        ValidationResult goalCheck = Validator.validateOptionalGoal(goalText);
        if (!goalCheck.isValid()) {
            message.setValue(goalCheck.getMessage());
            return;
        }

        float weight = weightCheck.getValue();
        busy.setValue(true);

        if (existingId == -1L) {
            weightRepository.add(weight, outcome -> afterWrite(outcome.isSuccess(),
                    outcome.getMessage(), weight, goalCheck));
        } else {
            weightRepository.update(existingId, weight, outcome -> afterWrite(outcome.isSuccess(),
                    outcome.getMessage(), weight, goalCheck));
        }
    }

    /**
     * Continues the save once the weight row has been written.
     *
     * @param success   whether the write succeeded
     * @param failure   the failure message when it did not
     * @param weight    the saved weight
     * @param goalCheck the validated goal field
     */
    private void afterWrite(boolean success, String failure, float weight,
                            ValidationResult goalCheck) {
        if (!success) {
            busy.setValue(false);
            message.setValue(failure);
            return;
        }

        if (goalCheck.hasValue()) {
            weightRepository.saveGoal(goalCheck.getValue(), goalOutcome -> {
                if (!goalOutcome.isSuccess()) {
                    message.setValue(goalOutcome.getMessage());
                }
                evaluateGoal(weight);
            });
        } else {
            evaluateGoal(weight);
        }
    }

    /**
     * Decides whether reaching the goal should raise a notification.
     *
     * Reads the preference through {@link SettingsPreferences}, which holds the one
     * default both screens use.
     *
     * @param weight the weight just saved
     */
    private void evaluateGoal(float weight) {
        weightRepository.getGoal(goal -> {
            busy.setValue(false);
            boolean reached = goal != null && weight <= goal;
            if (reached && settingsPreferences.isSmsEnabled()) {
                notifyGoalReached.setValue(true);
            } else {
                finished.setValue(true);
            }
        });
    }

    /** Called by the view once a message has been displayed. */
    public void onMessageHandled() {
        message.setValue(null);
    }

    /** Called by the view once the notification attempt has finished, successfully or not. */
    public void onNotificationHandled() {
        notifyGoalReached.setValue(false);
        finished.setValue(true);
    }
}
```

</details>

<details markdown="1">
<summary><code>SettingsViewModel.java</code></summary>

```java
package com.example.cs360project.viewmodel;

import androidx.lifecycle.LiveData;
import androidx.lifecycle.MutableLiveData;
import androidx.lifecycle.ViewModel;

import com.example.cs360project.data.SettingsPreferences;
import com.example.cs360project.data.WeightRepository;

/**
 * State and behavior for the settings screen.
 *
 * The delete runs through the repository and off the main thread. Both screens share
 * that repository, so the list picks the change up through LiveData.
 */
public class SettingsViewModel extends ViewModel {

    private final WeightRepository weightRepository;
    private final SettingsPreferences settingsPreferences;

    private final MutableLiveData<Boolean> smsEnabled = new MutableLiveData<>();
    private final MutableLiveData<String> message = new MutableLiveData<>();

    public SettingsViewModel(WeightRepository weightRepository,
                             SettingsPreferences settingsPreferences) {
        this.weightRepository = weightRepository;
        this.settingsPreferences = settingsPreferences;
        this.smsEnabled.setValue(settingsPreferences.isSmsEnabled());
    }

    /** Current state of the SMS opt in. */
    public LiveData<Boolean> getSmsEnabled() {
        return smsEnabled;
    }

    /** Messages to show the user. Cleared by {@link #onMessageHandled()}. */
    public LiveData<String> getMessage() {
        return message;
    }

    /**
     * Records the user's SMS choice.
     *
     * @param enabled true when the user has opted in and the permission is granted
     */
    public void setSmsEnabled(boolean enabled) {
        settingsPreferences.setSmsEnabled(enabled);
        smsEnabled.setValue(enabled);
    }

    /** Removes every stored weight entry. */
    public void deleteAllWeights() {
        weightRepository.deleteAll(outcome -> {
            int removed = outcome.getValue() == null ? 0 : outcome.getValue();
            message.setValue(removed == 0
                    ? "There was no weight data to delete"
                    : "Deleted " + removed + " weight " + (removed == 1 ? "entry" : "entries"));
        });
    }

    /** Called by the view once a message has been displayed. */
    public void onMessageHandled() {
        message.setValue(null);
    }
}
```

</details>

<details markdown="1">
<summary><code>ViewModelFactory.java</code></summary>

```java
package com.example.cs360project.viewmodel;

import android.content.Context;

import androidx.annotation.NonNull;
import androidx.lifecycle.ViewModel;
import androidx.lifecycle.ViewModelProvider;

import com.example.cs360project.data.ServiceLocator;
import com.example.cs360project.security.SessionManager;

/**
 * Supplies ViewModels with their dependencies.
 *
 * ViewModels take repositories through their constructors so a test can substitute fakes.
 * That means the framework cannot construct them with a no argument constructor, and this
 * factory is what bridges the two.
 */
public class ViewModelFactory implements ViewModelProvider.Factory {

    private final Context appContext;

    public ViewModelFactory(Context context) {
        this.appContext = context.getApplicationContext();
    }

    @NonNull
    @Override
    @SuppressWarnings("unchecked")
    public <T extends ViewModel> T create(@NonNull Class<T> modelClass) {
        if (modelClass.isAssignableFrom(LoginViewModel.class)) {
            return (T) new LoginViewModel(
                    ServiceLocator.userRepository(appContext),
                    SessionManager.getInstance());
        }
        if (modelClass.isAssignableFrom(WeightListViewModel.class)) {
            return (T) new WeightListViewModel(
                    ServiceLocator.weightRepository(appContext));
        }
        if (modelClass.isAssignableFrom(AddWeightViewModel.class)) {
            return (T) new AddWeightViewModel(
                    ServiceLocator.weightRepository(appContext),
                    ServiceLocator.settingsPreferences(appContext));
        }
        if (modelClass.isAssignableFrom(SettingsViewModel.class)) {
            return (T) new SettingsViewModel(
                    ServiceLocator.weightRepository(appContext),
                    ServiceLocator.settingsPreferences(appContext));
        }
        throw new IllegalArgumentException("Unknown ViewModel class: " + modelClass.getName());
    }
}
```

</details>

## Analytics

<details markdown="1">
<summary><code>WeightAnalytics.java</code></summary>

```java
package com.example.cs360project.analytics;

/**
 * Averages, extremes, streaks, and trend projection over a weight series.
 *
 * Pure functions, no Android imports, so the whole file is JVM testable. SQLite window
 * functions could do the moving average in the query and save pulling rows into memory,
 * but then it needs a device to test. The series is small either way.
 *
 * Indexed by position, not by date. See {@link GoalProjection} for why.
 */
public final class WeightAnalytics {

    private WeightAnalytics() {
        // Utility class, not instantiable.
    }

    /**
     * Moving average over a trailing window, in a single pass.
     *
     * A running sum adjusted by the entering and leaving element, so the whole series
     * costs O(n) rather than O(n times k).
     *
     * Positions before the window fills average over whatever exists, so the result is the
     * same length as the input.
     *
     * @param values the series in order, may be empty
     * @param window number of trailing entries to average over, at least 1
     * @return an array the same length as the input
     * @throws IllegalArgumentException if window is less than 1
     */
    public static float[] movingAverage(float[] values, int window) {
        if (window < 1) {
            throw new IllegalArgumentException("Window must be at least 1");
        }
        if (values == null || values.length == 0) {
            return new float[0];
        }

        float[] averages = new float[values.length];
        double runningSum = 0d;

        for (int i = 0; i < values.length; i++) {
            runningSum += values[i];
            if (i >= window) {
                runningSum -= values[i - window];
            }
            int count = Math.min(i + 1, window);
            averages[i] = (float) (runningSum / count);
        }
        return averages;
    }

    /**
     * Finds runs of consecutive entries moving the same direction.
     *
     * One pass, comparing each entry with the one before it. An equal reading ends both
     * runs.
     *
     * @param values the series in order, may be empty
     * @return the summary, never null
     */
    public static StreakSummary detectStreaks(float[] values) {
        if (values == null || values.length == 0) {
            return StreakSummary.empty();
        }
        if (values.length == 1) {
            return new StreakSummary(StreakSummary.Direction.FLAT, 1, 0, 0);
        }

        int longestDown = 0;
        int longestUp = 0;
        int runSteps = 0;
        StreakSummary.Direction runDirection = StreakSummary.Direction.FLAT;

        for (int i = 1; i < values.length; i++) {
            StreakSummary.Direction step;
            if (values[i] < values[i - 1]) {
                step = StreakSummary.Direction.DOWN;
            } else if (values[i] > values[i - 1]) {
                step = StreakSummary.Direction.UP;
            } else {
                step = StreakSummary.Direction.FLAT;
            }

            if (step == runDirection && step != StreakSummary.Direction.FLAT) {
                runSteps++;
            } else {
                runDirection = step;
                runSteps = step == StreakSummary.Direction.FLAT ? 0 : 1;
            }

            int runEntries = runSteps + 1;
            if (runDirection == StreakSummary.Direction.DOWN) {
                longestDown = Math.max(longestDown, runEntries);
            } else if (runDirection == StreakSummary.Direction.UP) {
                longestUp = Math.max(longestUp, runEntries);
            }
        }

        int currentLength = runDirection == StreakSummary.Direction.FLAT ? 1 : runSteps + 1;
        return new StreakSummary(runDirection, currentLength, longestDown, longestUp);
    }

    /**
     * Fits a line to the most recent entries and projects it toward a goal.
     *
     * Least squares from running sums of x, y, x squared and x times y, so one pass and
     * no stored deviations.
     *
     * A flat or wrong-direction trend gives no entry count. The goal is an upper bound,
     * matching the notification rule of firing at or below the goal.
     *
     * @param values      the series in order
     * @param recentCount how many trailing entries to fit, at least 2
     * @param goal        the target weight, treated as an upper bound
     * @return the projection, never null
     */
    public static GoalProjection projectToGoal(float[] values, int recentCount, float goal) {
        if (values == null || values.length < 2) {
            return GoalProjection.unavailable("Needs at least two entries");
        }
        if (recentCount < 2) {
            return GoalProjection.unavailable("Needs a window of at least two entries");
        }

        int n = Math.min(recentCount, values.length);
        int start = values.length - n;

        double sumX = 0d;
        double sumY = 0d;
        double sumXX = 0d;
        double sumXY = 0d;

        for (int i = 0; i < n; i++) {
            double x = i;
            double y = values[start + i];
            sumX += x;
            sumY += y;
            sumXX += x * x;
            sumXY += x * y;
        }

        double denominator = n * sumXX - sumX * sumX;
        if (denominator == 0d) {
            return GoalProjection.unavailable("Not enough distinct entries to fit a trend");
        }

        double slope = (n * sumXY - sumX * sumY) / denominator;
        double intercept = (sumY - slope * sumX) / n;
        double lastX = n - 1;
        double projectedNext = intercept + slope * (lastX + 1);
        float current = values[values.length - 1];

        // The goal is an upper bound, which is the same rule the notification uses: it
        // fires when the logged weight is at or below the goal. Treating it as a target to
        // hit exactly would report a user who has gone past their goal as moving away
        // from it.
        if (current <= goal) {
            return GoalProjection.of((float) slope, (float) projectedNext, 0f,
                    "Goal already reached");
        }

        if (Math.abs(slope) < 1e-6d) {
            return GoalProjection.of((float) slope, (float) projectedNext, null,
                    "Trend is flat, so the goal is not in reach on this pace");
        }
        if (slope > 0d) {
            return GoalProjection.of((float) slope, (float) projectedNext, null,
                    "Current trend moves away from the goal");
        }

        double xAtGoal = (goal - intercept) / slope;
        double entriesAway = xAtGoal - lastX;
        if (entriesAway <= 0d) {
            return GoalProjection.of((float) slope, (float) projectedNext, 0f,
                    "Goal already reached on the fitted trend");
        }
        return GoalProjection.of((float) slope, (float) projectedNext, (float) entriesAway,
                "On the recent trend the goal is about "
                        + Math.round(entriesAway) + " entries away");
    }
}
```

</details>

<details markdown="1">
<summary><code>MonotonicWindow.java</code></summary>

```java
package com.example.cs360project.analytics;

import java.util.ArrayDeque;
import java.util.Deque;

/**
 * Sliding window minimum and maximum without rescanning the window.
 *
 * Two deques of indices, each kept monotonic so the front is always the answer for the
 * current window. On append, indices at the back holding a worse value are dropped: a
 * value that is both worse and older can never win again. Each index is pushed once and
 * popped at most once, so the work per element does not depend on the window size.
 *
 * Append only. Removing from the middle could make a dropped index relevant again, and
 * the dropping is where the saving comes from. {@link AnalyticsEngine} rebuilds instead.
 */
public final class MonotonicWindow {

    private final int windowSize;
    private final Deque<Integer> minIndices = new ArrayDeque<>();
    private final Deque<Integer> maxIndices = new ArrayDeque<>();
    private final java.util.List<Float> values = new java.util.ArrayList<>();

    private long operationCount;

    /**
     * @param windowSize number of trailing entries the window covers, at least 1
     * @throws IllegalArgumentException if the window size is less than 1
     */
    public MonotonicWindow(int windowSize) {
        if (windowSize < 1) {
            throw new IllegalArgumentException("Window size must be at least 1");
        }
        this.windowSize = windowSize;
    }

    /**
     * Adds one value to the end of the series.
     *
     * @param value the value to append
     */
    public void append(float value) {
        int index = values.size();
        values.add(value);

        // Discard indices that can never be the minimum again. Each of these was pushed
        // once and is popped once, so the total work stays bounded.
        while (!minIndices.isEmpty() && values.get(minIndices.peekLast()) >= value) {
            minIndices.pollLast();
            operationCount++;
        }
        minIndices.addLast(index);
        operationCount++;

        while (!maxIndices.isEmpty() && values.get(maxIndices.peekLast()) <= value) {
            maxIndices.pollLast();
            operationCount++;
        }
        maxIndices.addLast(index);
        operationCount++;

        // Drop the front once it falls outside the window.
        int oldest = index - windowSize + 1;
        if (!minIndices.isEmpty() && minIndices.peekFirst() < oldest) {
            minIndices.pollFirst();
            operationCount++;
        }
        if (!maxIndices.isEmpty() && maxIndices.peekFirst() < oldest) {
            maxIndices.pollFirst();
            operationCount++;
        }
    }

    /**
     * @return the smallest value in the current window
     * @throws IllegalStateException if nothing has been appended yet
     */
    public float currentMin() {
        requireData();
        return values.get(minIndices.peekFirst());
    }

    /**
     * @return the largest value in the current window
     * @throws IllegalStateException if nothing has been appended yet
     */
    public float currentMax() {
        requireData();
        return values.get(maxIndices.peekFirst());
    }

    /**
     * @return how many values have been appended
     */
    public int size() {
        return values.size();
    }

    /**
     * @return the configured window size
     */
    public int windowSize() {
        return windowSize;
    }

    /**
     * Total deque pushes and pops performed since construction.
     *
     * Exposed so a test can assert the linear bound directly. A scan based implementation
     * would grow with window size here; this one does not.
     *
     * @return the running operation count
     */
    public long operationCount() {
        return operationCount;
    }

    private void requireData() {
        if (values.isEmpty()) {
            throw new IllegalStateException("No values have been appended");
        }
    }
}
```

</details>

<details markdown="1">
<summary><code>AnalyticsEngine.java</code></summary>

```java
package com.example.cs360project.analytics;

import java.util.ArrayList;
import java.util.List;

/**
 * Keeps the analytics current as the log changes.
 *
 * {@link #append(float)} handles a new entry with a fixed amount of work: one addition and
 * one subtraction on the running sum, an amortized constant update to the window, and one
 * comparison for the streak. Nothing there grows with the length of the log.
 *
 * {@link #rebuild(List)} handles an edit or a delete by replaying the series, which is
 * O(n). The window cannot have an element pulled out of its middle, so patching it is not
 * an option and rebuilding is the simpler answer. Edits are rare next to appends.
 *
 * Rebuild replays through append, so the two paths cannot drift apart.
 */
public final class AnalyticsEngine {

    /** Trailing entries covered by the average, minimum, and maximum. */
    public static final int DEFAULT_WINDOW_ENTRIES = 30;

    /** Trailing entries used to fit the goal trend. */
    public static final int DEFAULT_TREND_ENTRIES = 14;

    private final int windowSize;
    private final int trendSize;

    private final List<Float> values = new ArrayList<>();
    private MonotonicWindow window;
    private double windowSum;

    private StreakSummary.Direction runDirection = StreakSummary.Direction.FLAT;
    private int runSteps;
    private int longestDown;
    private int longestUp;

    private Float goal;

    public AnalyticsEngine() {
        this(DEFAULT_WINDOW_ENTRIES, DEFAULT_TREND_ENTRIES);
    }

    /**
     * @param windowSize trailing entries for the average, minimum, and maximum
     * @param trendSize  trailing entries used to fit the goal trend
     */
    public AnalyticsEngine(int windowSize, int trendSize) {
        this.windowSize = windowSize;
        this.trendSize = trendSize;
        this.window = new MonotonicWindow(windowSize);
    }

    /**
     * Sets the goal the projection aims at.
     *
     * @param goal the target weight, or null when none is set
     */
    public void setGoal(Float goal) {
        this.goal = goal;
    }

    /**
     * Adds one entry to the end of the series.
     *
     * Constant work with respect to the length of the log.
     *
     * @param value the new weight
     */
    public void append(float value) {
        int previousIndex = values.size() - 1;

        if (previousIndex >= 0) {
            float previous = values.get(previousIndex);
            StreakSummary.Direction step;
            if (value < previous) {
                step = StreakSummary.Direction.DOWN;
            } else if (value > previous) {
                step = StreakSummary.Direction.UP;
            } else {
                step = StreakSummary.Direction.FLAT;
            }

            if (step == runDirection && step != StreakSummary.Direction.FLAT) {
                runSteps++;
            } else {
                runDirection = step;
                runSteps = step == StreakSummary.Direction.FLAT ? 0 : 1;
            }

            int runEntries = runSteps + 1;
            if (runDirection == StreakSummary.Direction.DOWN) {
                longestDown = Math.max(longestDown, runEntries);
            } else if (runDirection == StreakSummary.Direction.UP) {
                longestUp = Math.max(longestUp, runEntries);
            }
        } else {
            runDirection = StreakSummary.Direction.FLAT;
            runSteps = 0;
        }

        values.add(value);
        window.append(value);

        windowSum += value;
        int leavingIndex = values.size() - windowSize - 1;
        if (leavingIndex >= 0) {
            windowSum -= values.get(leavingIndex);
        }
    }

    /**
     * Discards all state and replays the series.
     *
     * Called when an entry is edited or deleted. O(n) in the length of the series.
     *
     * @param series the full series in order, may be null or empty
     */
    public void rebuild(List<Float> series) {
        values.clear();
        window = new MonotonicWindow(windowSize);
        windowSum = 0d;
        runDirection = StreakSummary.Direction.FLAT;
        runSteps = 0;
        longestDown = 0;
        longestUp = 0;

        if (series == null) {
            return;
        }
        for (Float value : series) {
            if (value != null) {
                append(value);
            }
        }
    }

    /**
     * @return the current figures as one immutable value
     */
    public AnalyticsSnapshot snapshot() {
        if (values.isEmpty()) {
            return AnalyticsSnapshot.empty(windowSize);
        }

        int count = values.size();
        int covered = Math.min(count, windowSize);
        float average = (float) (windowSum / covered);
        float latest = values.get(count - 1);

        int currentLength = runDirection == StreakSummary.Direction.FLAT ? 1 : runSteps + 1;
        StreakSummary streaks = new StreakSummary(
                count == 1 ? StreakSummary.Direction.FLAT : runDirection,
                count == 1 ? 1 : currentLength,
                longestDown,
                longestUp);

        GoalProjection projection = goal == null
                ? GoalProjection.unavailable("No goal set")
                : WeightAnalytics.projectToGoal(toArray(), trendSize, goal);

        return new AnalyticsSnapshot(count, windowSize, latest, average,
                window.currentMin(), window.currentMax(), streaks, projection);
    }

    /**
     * @return the full moving average series, for charting or inspection
     */
    public float[] movingAverageSeries() {
        return WeightAnalytics.movingAverage(toArray(), windowSize);
    }

    /**
     * @return how many entries the engine currently holds
     */
    public int size() {
        return values.size();
    }

    /**
     * Deque operations performed by the window since the last rebuild.
     *
     * Exposed for the test that asserts the linear bound by counting.
     *
     * @return the running operation count
     */
    public long windowOperationCount() {
        return window.operationCount();
    }

    private float[] toArray() {
        float[] array = new float[values.size()];
        for (int i = 0; i < array.length; i++) {
            array[i] = values.get(i);
        }
        return array;
    }
}
```

</details>

<details markdown="1">
<summary><code>AnalyticsSnapshot.java</code></summary>

```java
package com.example.cs360project.analytics;

/**
 * Everything the analytics layer currently knows, as one immutable value.
 *
 * Published as a unit so a screen observing it never sees a half-updated set of figures,
 * for example a new minimum next to a stale average.
 */
public final class AnalyticsSnapshot {

    private final int entryCount;
    private final int windowSize;
    private final float latestValue;
    private final float movingAverage;
    private final float windowMin;
    private final float windowMax;
    private final StreakSummary streaks;
    private final GoalProjection projection;

    public AnalyticsSnapshot(int entryCount, int windowSize, float latestValue,
                             float movingAverage, float windowMin, float windowMax,
                             StreakSummary streaks, GoalProjection projection) {
        this.entryCount = entryCount;
        this.windowSize = windowSize;
        this.latestValue = latestValue;
        this.movingAverage = movingAverage;
        this.windowMin = windowMin;
        this.windowMax = windowMax;
        this.streaks = streaks;
        this.projection = projection;
    }

    /** Snapshot for a series with no entries. */
    public static AnalyticsSnapshot empty(int windowSize) {
        return new AnalyticsSnapshot(0, windowSize, 0f, 0f, 0f, 0f,
                StreakSummary.empty(), GoalProjection.unavailable("No entries yet"));
    }

    public boolean hasData() {
        return entryCount > 0;
    }

    public int getEntryCount() {
        return entryCount;
    }

    /** Number of trailing entries the average, minimum, and maximum cover. */
    public int getWindowSize() {
        return windowSize;
    }

    public float getLatestValue() {
        return latestValue;
    }

    /** Average over the trailing window, or over everything when the window is not yet full. */
    public float getMovingAverage() {
        return movingAverage;
    }

    public float getWindowMin() {
        return windowMin;
    }

    public float getWindowMax() {
        return windowMax;
    }

    public StreakSummary getStreaks() {
        return streaks;
    }

    public GoalProjection getProjection() {
        return projection;
    }
}
```

</details>

<details markdown="1">
<summary><code>StreakSummary.java</code></summary>

```java
package com.example.cs360project.analytics;

/**
 * Result of streak detection over a weight series.
 *
 * A streak is a run of consecutive entries moving the same way. Direction is judged one
 * step at a time: an entry lower than the one before it is a downward step, higher is an
 * upward step, and equal breaks both runs.
 *
 * Lengths are counted in entries, so two entries that fall in sequence are a streak of 2.
 */
public final class StreakSummary {

    /** Which way the most recent run is moving. */
    public enum Direction {
        DOWN, UP, FLAT, NONE
    }

    private final Direction currentDirection;
    private final int currentLength;
    private final int longestDownward;
    private final int longestUpward;

    public StreakSummary(Direction currentDirection, int currentLength,
                         int longestDownward, int longestUpward) {
        this.currentDirection = currentDirection;
        this.currentLength = currentLength;
        this.longestDownward = longestDownward;
        this.longestUpward = longestUpward;
    }

    /** Empty result for a series with nothing in it. */
    public static StreakSummary empty() {
        return new StreakSummary(Direction.NONE, 0, 0, 0);
    }

    public Direction getCurrentDirection() {
        return currentDirection;
    }

    /** Length of the run ending at the most recent entry, in entries. */
    public int getCurrentLength() {
        return currentLength;
    }

    /** Longest downward run anywhere in the series, in entries. */
    public int getLongestDownward() {
        return longestDownward;
    }

    /** Longest upward run anywhere in the series, in entries. */
    public int getLongestUpward() {
        return longestUpward;
    }
}
```

</details>

<details markdown="1">
<summary><code>GoalProjection.java</code></summary>

```java
package com.example.cs360project.analytics;

/**
 * Result of projecting a weight series forward toward a goal.
 *
 * Measured in entries, not days. The fit gives a slope per entry, and converting that to
 * a slope per day needs timestamps the series does not carry here.
 */
public final class GoalProjection {

    private final boolean available;
    private final float slopePerEntry;
    private final float projectedNextValue;
    private final Float entriesToGoal;
    private final String note;

    private GoalProjection(boolean available, float slopePerEntry, float projectedNextValue,
                           Float entriesToGoal, String note) {
        this.available = available;
        this.slopePerEntry = slopePerEntry;
        this.projectedNextValue = projectedNextValue;
        this.entriesToGoal = entriesToGoal;
        this.note = note;
    }

    /**
     * A projection that could not be computed.
     *
     * @param note why it is unavailable
     */
    public static GoalProjection unavailable(String note) {
        return new GoalProjection(false, 0f, 0f, null, note);
    }

    /**
     * A usable projection.
     *
     * @param slopePerEntry  fitted change in weight per entry, negative when trending down
     * @param projectedNext  fitted value one entry beyond the end of the series
     * @param entriesToGoal  entries until the fit crosses the goal, or null when it never does
     * @param note           short plain description of the outcome
     */
    public static GoalProjection of(float slopePerEntry, float projectedNext,
                                    Float entriesToGoal, String note) {
        return new GoalProjection(true, slopePerEntry, projectedNext, entriesToGoal, note);
    }

    /** False when the series was too short or too flat to fit a line. */
    public boolean isAvailable() {
        return available;
    }

    /** Fitted change per entry. Negative means the weight is trending down. */
    public float getSlopePerEntry() {
        return slopePerEntry;
    }

    /** Value the fit predicts for the next entry. */
    public float getProjectedNextValue() {
        return projectedNextValue;
    }

    /**
     * @return entries until the trend reaches the goal, or null when the trend never reaches it
     */
    public Float getEntriesToGoal() {
        return entriesToGoal;
    }

    /** Short plain description, suitable for display. */
    public String getNote() {
        return note;
    }
}
```

</details>

<details markdown="1">
<summary><code>AnalyticsBenchmark.java</code></summary>

```java
package com.example.cs360project.analytics;

import java.util.Locale;
import java.util.Random;

/**
 * Times the single pass and sliding window algorithms against the naive versions.
 *
 * A {@code main} rather than a unit test, since a timing assertion is flaky on a loaded
 * machine. The complexity claim is asserted by counting operations in
 * {@code MonotonicWindowTest}.
 *
 * Run with {@code java com.example.cs360project.analytics.AnalyticsBenchmark}.
 */
public final class AnalyticsBenchmark {

    private static final int WINDOW = 30;
    private static final int[] SIZES = {100, 10_000, 100_000};
    private static final int REPEATS = 5;

    private AnalyticsBenchmark() {
    }

    public static void main(String[] args) {
        System.out.printf(Locale.US, "Window size %d, best of %d runs, times in milliseconds%n%n",
                WINDOW, REPEATS);
        System.out.printf(Locale.US, "%-10s %14s %14s %10s %14s %14s %10s%n",
                "entries", "avg naive", "avg 1-pass", "speedup",
                "minmax scan", "minmax deque", "speedup");

        for (int n : SIZES) {
            float[] values = generate(n);

            double naiveAvg = bestOf(() -> naiveMovingAverage(values, WINDOW));
            double fastAvg = bestOf(() -> WeightAnalytics.movingAverage(values, WINDOW));
            double naiveExtremes = bestOf(() -> naiveWindowExtremes(values, WINDOW));
            double dequeExtremes = bestOf(() -> dequeWindowExtremes(values, WINDOW));

            System.out.printf(Locale.US, "%-10d %14.3f %14.3f %9.1fx %14.3f %14.3f %9.1fx%n",
                    n, naiveAvg, fastAvg, ratio(naiveAvg, fastAvg),
                    naiveExtremes, dequeExtremes, ratio(naiveExtremes, dequeExtremes));
        }

        System.out.println();
        System.out.println("Operation counts for the sliding window (pushes plus pops):");
        for (int n : SIZES) {
            float[] values = generate(n);
            MonotonicWindow window = new MonotonicWindow(WINDOW);
            for (float value : values) {
                window.append(value);
            }
            System.out.printf(Locale.US, "  %-8d entries  %,12d operations  %.2f per entry%n",
                    n, window.operationCount(), (double) window.operationCount() / n);
        }

        windowSweep();
    }

    /**
     * Times both window implementations as the window widens on a fixed series.
     *
     * This is the measurement that actually shows the difference. Holding the number of
     * entries constant and widening the window leaves the deque flat, because its cost
     * depends on the number of elements and not on how many of them the window covers,
     * while the rescan grows in proportion to the width. The two cross at a window of
     * roughly thirty, below which the rescan is competitive: scanning a couple of dozen
     * floats in a row is cache friendly, and the deque pays for boxing its indices.
     */
    private static void windowSweep() {
        int n = 100_000;
        float[] values = generate(n);

        System.out.println();
        System.out.printf(Locale.US,
                "Sliding minimum and maximum over %,d entries as the window widens:%n", n);
        System.out.printf(Locale.US, "%-10s %14s %14s %10s%n",
                "window", "rescan", "deque", "speedup");

        for (int window : new int[]{7, 30, 90, 365, 2000, 10_000}) {
            double scan = bestOf(() -> naiveWindowExtremes(values, window));
            double deque = bestOf(() -> dequeWindowExtremes(values, window));
            System.out.printf(Locale.US, "%-10d %14.2f %14.2f %9.1fx%n",
                    window, scan, deque, ratio(scan, deque));
        }
    }

    private static float[] generate(int n) {
        Random random = new Random(20260927L);
        float[] values = new float[n];
        float weight = 210f;
        for (int i = 0; i < n; i++) {
            weight += (random.nextFloat() - 0.52f) * 1.5f;
            values[i] = weight;
        }
        return values;
    }

    private static double bestOf(Runnable work) {
        double best = Double.MAX_VALUE;
        for (int i = 0; i < REPEATS; i++) {
            long start = System.nanoTime();
            work.run();
            double elapsed = (System.nanoTime() - start) / 1_000_000d;
            best = Math.min(best, elapsed);
        }
        return best;
    }

    private static double ratio(double slow, double fast) {
        return fast <= 0d ? 0d : slow / fast;
    }

    /** The O(n times k) moving average the running sum replaces. */
    private static float[] naiveMovingAverage(float[] values, int window) {
        float[] result = new float[values.length];
        for (int i = 0; i < values.length; i++) {
            int start = Math.max(0, i - window + 1);
            double sum = 0d;
            for (int j = start; j <= i; j++) {
                sum += values[j];
            }
            result[i] = (float) (sum / (i - start + 1));
        }
        return result;
    }

    /** The rescan-every-window minimum and maximum the deque replaces. */
    private static float[] naiveWindowExtremes(float[] values, int window) {
        float[] result = new float[values.length * 2];
        for (int i = 0; i < values.length; i++) {
            int start = Math.max(0, i - window + 1);
            float min = Float.MAX_VALUE;
            float max = -Float.MAX_VALUE;
            for (int j = start; j <= i; j++) {
                min = Math.min(min, values[j]);
                max = Math.max(max, values[j]);
            }
            result[i * 2] = min;
            result[i * 2 + 1] = max;
        }
        return result;
    }

    private static float[] dequeWindowExtremes(float[] values, int window) {
        float[] result = new float[values.length * 2];
        MonotonicWindow monotonic = new MonotonicWindow(window);
        for (int i = 0; i < values.length; i++) {
            monotonic.append(values[i]);
            result[i * 2] = monotonic.currentMin();
            result[i * 2 + 1] = monotonic.currentMax();
        }
        return result;
    }
}
```

</details>

## Data layer

<details markdown="1">
<summary><code>AppDatabase.java</code></summary>

```java
package com.example.cs360project.data;

import android.content.Context;

import androidx.room.Database;
import androidx.room.Room;
import androidx.room.RoomDatabase;

import com.example.cs360project.data.dao.RoomUserDao;
import com.example.cs360project.data.dao.RoomWeightDao;
import com.example.cs360project.data.entity.GoalEntity;
import com.example.cs360project.data.entity.UserEntity;
import com.example.cs360project.data.entity.WeightEntryEntity;
import com.example.cs360project.data.migration.Migrations;

/**
 * The Room database.
 *
 * Room verifies every query against the schema at build time, so a column renamed in an
 * entity and missed in a query fails the build.
 *
 * Version 2, with an explicit migration from 1. Do not add
 * fallbackToDestructiveMigration to get past a schema change: it wipes the database, and
 * a missing migration should fail loudly in development instead.
 *
 * Foreign keys are enforced at open time. SQLite leaves them off by default.
 */
@Database(
        entities = {UserEntity.class, WeightEntryEntity.class, GoalEntity.class},
        version = 2,
        exportSchema = true)
public abstract class AppDatabase extends RoomDatabase {

    private static final String DATABASE_NAME = "weight_tracker.db";

    private static volatile AppDatabase instance;

    public abstract RoomUserDao userDao();

    public abstract RoomWeightDao weightDao();

    /**
     * Opens the database, running the migration when an older file is present.
     *
     * The file name matches the one version 1 used, so existing data is found and
     * migrated.
     *
     * @param context any context; the application context is retained
     * @return the shared database
     */
    public static AppDatabase getInstance(Context context) {
        if (instance == null) {
            synchronized (AppDatabase.class) {
                if (instance == null) {
                    instance = Room.databaseBuilder(
                                    context.getApplicationContext(),
                                    AppDatabase.class,
                                    DATABASE_NAME)
                            .addMigrations(Migrations.MIGRATION_1_2)
                            .build();
                }
            }
        }
        return instance;
    }
}
```

</details>

<details markdown="1">
<summary><code>WeightRepository.java</code></summary>

```java
package com.example.cs360project.data;

import androidx.lifecycle.LiveData;
import androidx.lifecycle.MutableLiveData;

import com.example.cs360project.analytics.AnalyticsEngine;
import com.example.cs360project.analytics.AnalyticsSnapshot;
import com.example.cs360project.data.dao.WeightDao;
import com.example.cs360project.data.model.WeightItem;
import com.example.cs360project.data.model.WeightStats;
import com.example.cs360project.security.SessionManager;
import com.example.cs360project.util.AppExecutors;
import com.example.cs360project.util.Callback;
import com.example.cs360project.util.Outcome;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

/**
 * The only path to stored weights, and to the figures derived from them.
 *
 * Every call carries the signed in user id, which scopes the query to its owner.
 *
 * Derived figures come from two places. The database computes the count, extremes, mean
 * and span over an index. The moving average, sliding window, streaks and projection stay
 * in Java, since each result depends on the one before it.
 *
 * {@link #add} appends to the analytics engine, which is constant work. Edits and deletes
 * rebuild it, since the window cannot have an element pulled out of its middle.
 *
 * Everything runs on the background executor and is posted back.
 */
public class WeightRepository {

    private final WeightDao weightDao;
    private final AppExecutors executors;
    private final SessionManager session;
    private final AnalyticsEngine analytics = new AnalyticsEngine();

    private final MutableLiveData<List<WeightItem>> weights = new MutableLiveData<>();
    private final MutableLiveData<AnalyticsSnapshot> analyticsSnapshot = new MutableLiveData<>();
    private final MutableLiveData<WeightStats> stats = new MutableLiveData<>();

    public WeightRepository(WeightDao weightDao, AppExecutors executors,
                            SessionManager session) {
        this.weightDao = weightDao;
        this.executors = executors;
        this.session = session;
        this.weights.setValue(Collections.emptyList());
        this.analyticsSnapshot.setValue(
                AnalyticsSnapshot.empty(AnalyticsEngine.DEFAULT_WINDOW_ENTRIES));
        this.stats.setValue(new WeightStats());
    }

    /**
     * @return the signed in user's entries, in recorded order
     */
    public LiveData<List<WeightItem>> getWeights() {
        return weights;
    }

    /**
     * @return the figures computed in Java from the sequence
     */
    public LiveData<AnalyticsSnapshot> getAnalytics() {
        return analyticsSnapshot;
    }

    /**
     * @return the figures computed by the database
     */
    public LiveData<WeightStats> getStats() {
        return stats;
    }

    /** Reloads from storage and rebuilds the derived figures. */
    public void refresh() {
        executors.diskIO().execute(this::reloadAndRebuild);
    }

    /**
     * Adds an entry for the signed in user.
     *
     * @param weight   the validated value
     * @param callback invoked on the main thread with the outcome
     */
    public void add(float weight, Callback<Outcome<Void>> callback) {
        executors.diskIO().execute(() -> {
            long userId = session.getUserId();
            long id = weightDao.insert(userId, weight, System.currentTimeMillis());

            Outcome<Void> outcome;
            if (id == -1L) {
                outcome = Outcome.failure("Could not save the entry");
            } else {
                outcome = Outcome.success(null);
                analytics.append(weight);
            }
            weights.postValue(weightDao.findAll(userId));
            analyticsSnapshot.postValue(analytics.snapshot());
            stats.postValue(weightDao.stats(userId));
            deliver(outcome, callback);
        });
    }

    /**
     * Updates one of the signed in user's entries.
     *
     * A row belonging to anyone else matches nothing and reports a failure.
     *
     * @param id       the row to change
     * @param weight   the validated value
     * @param callback invoked on the main thread with the outcome
     */
    public void update(long id, float weight, Callback<Outcome<Void>> callback) {
        executors.diskIO().execute(() -> {
            int changed = weightDao.update(
                    id, session.getUserId(), weight, System.currentTimeMillis());
            Outcome<Void> outcome = changed == 0
                    ? Outcome.failure("Could not update the entry")
                    : Outcome.success(null);
            reloadAndRebuild();
            deliver(outcome, callback);
        });
    }

    /**
     * Deletes one of the signed in user's entries.
     *
     * @param id       the row to remove
     * @param callback invoked on the main thread with the outcome
     */
    public void delete(long id, Callback<Outcome<Void>> callback) {
        executors.diskIO().execute(() -> {
            int removed = weightDao.delete(id, session.getUserId());
            Outcome<Void> outcome = removed == 0
                    ? Outcome.failure("Could not delete the entry")
                    : Outcome.success(null);
            reloadAndRebuild();
            deliver(outcome, callback);
        });
    }

    /**
     * Deletes every entry belonging to the signed in user.
     *
     * @param callback invoked on the main thread with the number of rows removed
     */
    public void deleteAll(Callback<Outcome<Integer>> callback) {
        executors.diskIO().execute(() -> {
            int removed = weightDao.deleteAll(session.getUserId());
            reloadAndRebuild();
            executors.mainThread().execute(() -> callback.onResult(Outcome.success(removed)));
        });
    }

    /**
     * Reads the signed in user's goal weight.
     *
     * @param callback invoked on the main thread with the goal, or null when none is set
     */
    public void getGoal(Callback<Float> callback) {
        executors.diskIO().execute(() -> {
            Float goal = weightDao.findGoal(session.getUserId());
            executors.mainThread().execute(() -> callback.onResult(goal));
        });
    }

    /**
     * Replaces the signed in user's goal weight.
     *
     * @param goal     the validated value
     * @param callback invoked on the main thread with the outcome
     */
    public void saveGoal(float goal, Callback<Outcome<Void>> callback) {
        executors.diskIO().execute(() -> {
            boolean saved = weightDao.saveGoal(session.getUserId(), goal);
            if (saved) {
                analytics.setGoal(goal);
                analyticsSnapshot.postValue(analytics.snapshot());
            }
            Outcome<Void> outcome = saved
                    ? Outcome.success(null)
                    : Outcome.failure("Could not save the goal");
            deliver(outcome, callback);
        });
    }

    /**
     * Summary figures over a time range, straight from the database.
     *
     * Equality on userId then a range on recordedAt, which is the order the composite
     * index is built in.
     *
     * @param fromInclusive epoch milliseconds
     * @param toInclusive   epoch milliseconds
     * @param callback      invoked on the main thread with the figures
     */
    public void statsBetween(long fromInclusive, long toInclusive,
                             Callback<WeightStats> callback) {
        executors.diskIO().execute(() -> {
            WeightStats range = weightDao.statsBetween(
                    session.getUserId(), fromInclusive, toInclusive);
            executors.mainThread().execute(() -> callback.onResult(range));
        });
    }

    /**
     * Loads the series and replays it through the engine.
     *
     * Called from the background executor only.
     */
    private void reloadAndRebuild() {
        long userId = session.getUserId();
        List<WeightItem> items = weightDao.findAll(userId);

        List<Float> series = new ArrayList<>(items.size());
        for (WeightItem item : items) {
            series.add(item.getWeight());
        }

        analytics.setGoal(weightDao.findGoal(userId));
        analytics.rebuild(series);

        weights.postValue(items);
        analyticsSnapshot.postValue(analytics.snapshot());
        stats.postValue(weightDao.stats(userId));
    }

    /**
     * Delivers an outcome on the main thread.
     *
     * Called from the background executor only.
     */
    private void deliver(Outcome<Void> outcome, Callback<Outcome<Void>> callback) {
        executors.mainThread().execute(() -> callback.onResult(outcome));
    }
}
```

</details>

<details markdown="1">
<summary><code>UserRepository.java</code></summary>

```java
package com.example.cs360project.data;

import com.example.cs360project.data.dao.UserDao;
import com.example.cs360project.data.model.AuthenticatedUser;
import com.example.cs360project.data.model.User;
import com.example.cs360project.security.PasswordHasher;
import com.example.cs360project.util.AppExecutors;
import com.example.cs360project.util.Callback;
import com.example.cs360project.util.Outcome;

/**
 * Registration and sign in.
 *
 * Sits between the DAO and the ViewModel, and owns threading, hashing, and what a failed
 * sign in is allowed to reveal.
 *
 * The DAO arrives through the constructor, so a test can pass a fake and run this on the
 * JVM.
 */
public class UserRepository {

    private final UserDao userDao;
    private final AppExecutors executors;

    public UserRepository(UserDao userDao, AppExecutors executors) {
        this.userDao = userDao;
        this.executors = executors;
    }

    /**
     * Creates an account.
     *
     * The password is hashed on the background thread and the caller's array is cleared
     * as soon as the hash exists. The plaintext never reaches the DAO.
     *
     * @param username the requested account name, already validated
     * @param password the password characters; this method clears them
     * @param callback invoked on the main thread with the outcome
     */
    public void register(String username, char[] password, Callback<Outcome<Void>> callback) {
        executors.diskIO().execute(() -> {
            Outcome<Void> outcome;
            try {
                if (userDao.findByUsername(username) != null) {
                    outcome = Outcome.failure("That username is already taken");
                } else {
                    String encoded = PasswordHasher.hash(password);
                    long id = userDao.insert(username, encoded);
                    outcome = id == -1
                            ? Outcome.failure("Could not create the account")
                            : Outcome.success(null);
                }
            } catch (RuntimeException e) {
                outcome = Outcome.failure("Could not create the account");
            } finally {
                PasswordHasher.wipe(password);
            }

            Outcome<Void> delivered = outcome;
            executors.mainThread().execute(() -> callback.onResult(delivered));
        });
    }

    /**
     * Authenticates a user.
     *
     * When no account matches the username, the password is still verified against an
     * unmatchable value. Skipping the verification would make a lookup for an unknown user
     * return noticeably faster than one for a known user, which tells an attacker which
     * usernames are registered. The failure message is the same either way for the same
     * reason.
     *
     * @param username the submitted account name
     * @param password the password characters; this method clears them
     * @param callback invoked on the main thread with the authenticated identity or a failure
     */
    public void login(String username, char[] password,
                      Callback<Outcome<AuthenticatedUser>> callback) {
        executors.diskIO().execute(() -> {
            Outcome<AuthenticatedUser> outcome;
            try {
                User stored = userDao.findByUsername(username);
                String encoded = stored == null
                        ? PasswordHasher.unmatchableHash()
                        : stored.getEncodedPassword();

                boolean matches = PasswordHasher.verify(password, encoded);

                outcome = (matches && stored != null)
                        ? Outcome.success(new AuthenticatedUser(stored.getId(), stored.getUsername()))
                        : Outcome.failure("Invalid username or password");
            } catch (RuntimeException e) {
                outcome = Outcome.failure("Invalid username or password");
            } finally {
                PasswordHasher.wipe(password);
            }

            Outcome<AuthenticatedUser> delivered = outcome;
            executors.mainThread().execute(() -> callback.onResult(delivered));
        });
    }
}
```

</details>

<details markdown="1">
<summary><code>SettingsPreferences.java</code></summary>

```java
package com.example.cs360project.data;

import android.content.Context;
import android.content.SharedPreferences;

/**
 * The only place the SMS preference is read or written.
 *
 * The key and the default are declared once here, so two call sites cannot disagree on
 * a fresh install before the key has been written.
 *
 * {@link #DEFAULT_SMS_ENABLED} is false, so SMS is opt in.
 */
public class SettingsPreferences {

    private static final String PREFS_NAME = "AppPrefs";
    private static final String KEY_SMS_ENABLED = "sms_enabled";

    /** Opt out by default. Both call sites read this same constant. */
    public static final boolean DEFAULT_SMS_ENABLED = false;

    private final SharedPreferences preferences;

    public SettingsPreferences(Context context) {
        this.preferences = context.getApplicationContext()
                .getSharedPreferences(PREFS_NAME, Context.MODE_PRIVATE);
    }

    /**
     * @return true when the user has opted into SMS notifications
     */
    public boolean isSmsEnabled() {
        return preferences.getBoolean(KEY_SMS_ENABLED, DEFAULT_SMS_ENABLED);
    }

    /**
     * @param enabled the new preference value
     */
    public void setSmsEnabled(boolean enabled) {
        preferences.edit().putBoolean(KEY_SMS_ENABLED, enabled).apply();
    }
}
```

</details>

<details markdown="1">
<summary><code>ServiceLocator.java</code></summary>

```java
package com.example.cs360project.data;

import android.content.Context;

import com.example.cs360project.data.dao.RoomUserDaoAdapter;
import com.example.cs360project.data.dao.RoomWeightDaoAdapter;
import com.example.cs360project.security.SessionManager;
import com.example.cs360project.util.AppExecutors;

/**
 * Builds and holds the single instance of each dependency.
 *
 * There has to be exactly one {@link WeightRepository}. The list and settings screens
 * observe the same LiveData, and a second repository would break that.
 *
 * This is the only place that decides what satisfies the DAO interfaces, which is now an
 * adapter over a Room DAO.
 */
public final class ServiceLocator {

    private static volatile WeightRepository weightRepository;
    private static volatile UserRepository userRepository;
    private static volatile SettingsPreferences settingsPreferences;

    private ServiceLocator() {
        // Static access only.
    }

    public static WeightRepository weightRepository(Context context) {
        if (weightRepository == null) {
            synchronized (ServiceLocator.class) {
                if (weightRepository == null) {
                    AppDatabase database = AppDatabase.getInstance(context);
                    weightRepository = new WeightRepository(
                            new RoomWeightDaoAdapter(database.weightDao()),
                            AppExecutors.getInstance(),
                            SessionManager.getInstance());
                }
            }
        }
        return weightRepository;
    }

    public static UserRepository userRepository(Context context) {
        if (userRepository == null) {
            synchronized (ServiceLocator.class) {
                if (userRepository == null) {
                    AppDatabase database = AppDatabase.getInstance(context);
                    userRepository = new UserRepository(
                            new RoomUserDaoAdapter(database.userDao()),
                            AppExecutors.getInstance());
                }
            }
        }
        return userRepository;
    }

    public static SettingsPreferences settingsPreferences(Context context) {
        if (settingsPreferences == null) {
            synchronized (ServiceLocator.class) {
                if (settingsPreferences == null) {
                    settingsPreferences = new SettingsPreferences(context);
                }
            }
        }
        return settingsPreferences;
    }
}
```

</details>

## Entities

<details markdown="1">
<summary><code>UserEntity.java</code></summary>

```java
package com.example.cs360project.data.entity;

import androidx.annotation.NonNull;
import androidx.room.ColumnInfo;
import androidx.room.Entity;
import androidx.room.Index;
import androidx.room.PrimaryKey;

/**
 * An account row.
 *
 * The unique index on username is what enforces uniqueness. A check in application code
 * loses to a race between two registrations.
 *
 * The credential occupies four columns. Having the algorithm and iteration count as
 * columns makes finding credentials on an old work factor a query.
 */
@Entity(tableName = "users",
        indices = {@Index(value = "username", unique = true)})
public class UserEntity {

    @PrimaryKey(autoGenerate = true)
    @ColumnInfo(name = "id")
    public long id;

    @NonNull
    @ColumnInfo(name = "username")
    public String username = "";

    /** Key derivation function identifier, for example pbkdf2_sha256. */
    @NonNull
    @ColumnInfo(name = "passwordAlgorithm")
    public String passwordAlgorithm = "";

    /** Work factor the stored hash was produced with. */
    @ColumnInfo(name = "passwordIterations")
    public int passwordIterations;

    /** Base64 salt, unique per credential. */
    @NonNull
    @ColumnInfo(name = "passwordSalt")
    public String passwordSalt = "";

    /** Base64 derived key. Never logged and never returned to the UI layer. */
    @NonNull
    @ColumnInfo(name = "passwordHash")
    public String passwordHash = "";
}
```

</details>

<details markdown="1">
<summary><code>WeightEntryEntity.java</code></summary>

```java
package com.example.cs360project.data.entity;

import androidx.room.ColumnInfo;
import androidx.room.Entity;
import androidx.room.ForeignKey;
import androidx.room.Index;
import androidx.room.PrimaryKey;

/**
 * One logged weight, owned by a user.
 *
 * The userId foreign key is what scopes a row to its owner. Cascading delete removes an
 * account's entries with it.
 *
 * recordedAt is epoch milliseconds, so it sorts chronologically, subtracts, and can carry
 * an index.
 *
 * The index covers userId and recordedAt in that order, matching the one access path the
 * application has: equality on the user, then a range on the time.
 */
@Entity(tableName = "weight_entries",
        foreignKeys = @ForeignKey(
                entity = UserEntity.class,
                parentColumns = "id",
                childColumns = "userId",
                onDelete = ForeignKey.CASCADE),
        indices = {@Index(value = {"userId", "recordedAt"})})
public class WeightEntryEntity {

    @PrimaryKey(autoGenerate = true)
    @ColumnInfo(name = "id")
    public long id;

    @ColumnInfo(name = "userId")
    public long userId;

    @ColumnInfo(name = "weight")
    public float weight;

    /** Epoch milliseconds. */
    @ColumnInfo(name = "recordedAt")
    public long recordedAt;
}
```

</details>

<details markdown="1">
<summary><code>GoalEntity.java</code></summary>

```java
package com.example.cs360project.data.entity;

import androidx.room.ColumnInfo;
import androidx.room.Entity;
import androidx.room.ForeignKey;
import androidx.room.Index;
import androidx.room.PrimaryKey;

/**
 * A user's goal weight.
 *
 * One row per user, enforced by a unique index on userId. That constraint is what lets
 * the goal be written with a single upsert.
 */
@Entity(tableName = "goals",
        foreignKeys = @ForeignKey(
                entity = UserEntity.class,
                parentColumns = "id",
                childColumns = "userId",
                onDelete = ForeignKey.CASCADE),
        indices = {@Index(value = "userId", unique = true)})
public class GoalEntity {

    @PrimaryKey(autoGenerate = true)
    @ColumnInfo(name = "id")
    public long id;

    @ColumnInfo(name = "userId")
    public long userId;

    @ColumnInfo(name = "goalWeight")
    public float goalWeight;
}
```

</details>

## Data access

<details markdown="1">
<summary><code>WeightDao.java</code></summary>

```java
package com.example.cs360project.data.dao;

import com.example.cs360project.data.model.WeightItem;
import com.example.cs360project.data.model.WeightStats;

import java.util.List;

/**
 * Storage operations for weight entries and goals.
 *
 * Every method takes the owning user, so the owner is part of identifying a row.
 *
 * The goal sits here. It is one row that only the logging flow touches.
 */
public interface WeightDao {

    /**
     * @return the user's entries in recorded order
     */
    List<WeightItem> findAll(long userId);

    /**
     * @param recordedAt epoch milliseconds
     * @return the new row id, or -1 on failure
     */
    long insert(long userId, float weight, long recordedAt);

    /**
     * @return the number of rows changed, which is 0 when the row belongs to someone else
     */
    int update(long id, long userId, float weight, long recordedAt);

    /**
     * @return the number of rows removed, which is 0 when the row belongs to someone else
     */
    int delete(long id, long userId);

    /**
     * @return the number of rows removed
     */
    int deleteAll(long userId);

    /**
     * @return the stored goal weight, or null when none has been set
     */
    Float findGoal(long userId);

    /**
     * @return true when the write succeeded
     */
    boolean saveGoal(long userId, float goal);

    /**
     * Summary figures over the whole series, computed by the database.
     *
     * @return the figures, never null
     */
    WeightStats stats(long userId);

    /**
     * Summary figures over a time range.
     *
     * @param fromInclusive epoch milliseconds
     * @param toInclusive   epoch milliseconds
     * @return the figures, never null
     */
    WeightStats statsBetween(long userId, long fromInclusive, long toInclusive);
}
```

</details>

<details markdown="1">
<summary><code>UserDao.java</code></summary>

```java
package com.example.cs360project.data.dao;

import com.example.cs360project.data.model.User;

/**
 * Storage operations for accounts.
 *
 * An interface so the statements behind it can be swapped without the repository
 * changing, and so a fake can stand in during unit tests.
 */
public interface UserDao {

    /**
     * Inserts a new account.
     *
     * @param username        the account name
     * @param encodedPassword the encoded salt and hash, never a plaintext password
     * @return the new row id, or -1 on failure
     */
    long insert(String username, String encodedPassword);

    /**
     * Looks up an account by name.
     *
     * @param username the account name
     * @return the stored row, or null when no account matches
     */
    User findByUsername(String username);
}
```

</details>

<details markdown="1">
<summary><code>RoomWeightDao.java</code></summary>

```java
package com.example.cs360project.data.dao;

import androidx.room.Dao;
import androidx.room.Insert;
import androidx.room.OnConflictStrategy;
import androidx.room.Query;

import com.example.cs360project.data.entity.GoalEntity;
import com.example.cs360project.data.entity.WeightEntryEntity;
import com.example.cs360project.data.model.WeightStats;

import java.util.List;

/**
 * Room queries for weight entries and goals.
 *
 * Room checks every one of these against the schema when the project is built, so a
 * mistyped column name is a compile error instead of something that throws on a device
 * once a particular screen is opened.
 *
 * Every statement names userId. A delete that matches only on row id will remove a row
 * belonging to somebody else if an id is guessed or reused. With the owner in the WHERE
 * clause the database refuses, so the caller does not have to remember to check.
 */
@Dao
public interface RoomWeightDao {

    @Query("SELECT * FROM weight_entries WHERE userId = :userId ORDER BY recordedAt, id")
    List<WeightEntryEntity> findAll(long userId);

    @Insert
    long insert(WeightEntryEntity entry);

    @Query("UPDATE weight_entries SET weight = :weight, recordedAt = :recordedAt "
            + "WHERE id = :id AND userId = :userId")
    int update(long id, long userId, float weight, long recordedAt);

    @Query("DELETE FROM weight_entries WHERE id = :id AND userId = :userId")
    int delete(long id, long userId);

    @Query("DELETE FROM weight_entries WHERE userId = :userId")
    int deleteAll(long userId);

    /**
     * Whole-series summary, computed in SQL.
     *
     * Returns one row. The index on userId and recordedAt covers the lookup.
     */
    @Query("SELECT COUNT(*) AS entryCount, "
            + "MIN(weight) AS minWeight, "
            + "MAX(weight) AS maxWeight, "
            + "AVG(weight) AS averageWeight, "
            + "MIN(recordedAt) AS firstRecordedAt, "
            + "MAX(recordedAt) AS lastRecordedAt "
            + "FROM weight_entries WHERE userId = :userId")
    WeightStats stats(long userId);

    /**
     * The same summary over a time range.
     *
     * This is the query the composite index was designed for: equality on userId, then a
     * range on recordedAt. Neither half works without the timestamp being an integer.
     */
    @Query("SELECT COUNT(*) AS entryCount, "
            + "MIN(weight) AS minWeight, "
            + "MAX(weight) AS maxWeight, "
            + "AVG(weight) AS averageWeight, "
            + "MIN(recordedAt) AS firstRecordedAt, "
            + "MAX(recordedAt) AS lastRecordedAt "
            + "FROM weight_entries "
            + "WHERE userId = :userId AND recordedAt BETWEEN :fromInclusive AND :toInclusive")
    WeightStats statsBetween(long userId, long fromInclusive, long toInclusive);

    @Query("SELECT goalWeight FROM goals WHERE userId = :userId LIMIT 1")
    Float findGoal(long userId);

    /**
     * Writes the goal, replacing the row that is already there.
     *
     * One statement, because the unique index on userId turns a second insert into a
     * replace. Delete followed by insert would leave a window with no goal stored at all.
     */
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    long upsertGoal(GoalEntity goal);
}
```

</details>

<details markdown="1">
<summary><code>RoomUserDao.java</code></summary>

```java
package com.example.cs360project.data.dao;

import androidx.room.Dao;
import androidx.room.Insert;
import androidx.room.OnConflictStrategy;
import androidx.room.Query;

import com.example.cs360project.data.entity.UserEntity;

/**
 * Room queries for accounts.
 *
 * The insert aborts on conflict, so a duplicate username fails the registration instead
 * of overwriting the existing credential.
 */
@Dao
public interface RoomUserDao {

    @Insert(onConflict = OnConflictStrategy.ABORT)
    long insert(UserEntity user);

    @Query("SELECT * FROM users WHERE username = :username LIMIT 1")
    UserEntity findByUsername(String username);

    @Query("SELECT COUNT(*) FROM users WHERE passwordIterations < :minimumIterations")
    int countCredentialsBelow(int minimumIterations);
}
```

</details>

<details markdown="1">
<summary><code>RoomWeightDaoAdapter.java</code></summary>

```java
package com.example.cs360project.data.dao;

import com.example.cs360project.data.entity.GoalEntity;
import com.example.cs360project.data.entity.WeightEntryEntity;
import com.example.cs360project.data.model.WeightItem;
import com.example.cs360project.data.model.WeightStats;

import java.util.ArrayList;
import java.util.List;

/**
 * Presents the Room DAO as the {@link WeightDao} the repository already expects.
 *
 * Room works in entities and generates its own implementation. The layers above work in
 * the model type and know nothing about either. Translating between them here meant the
 * repository, the ViewModels and the screens compiled unchanged when the storage engine
 * was swapped.
 */
public class RoomWeightDaoAdapter implements WeightDao {

    private final RoomWeightDao roomDao;

    public RoomWeightDaoAdapter(RoomWeightDao roomDao) {
        this.roomDao = roomDao;
    }

    @Override
    public List<WeightItem> findAll(long userId) {
        List<WeightEntryEntity> rows = roomDao.findAll(userId);
        List<WeightItem> items = new ArrayList<>(rows.size());
        for (WeightEntryEntity row : rows) {
            items.add(new WeightItem(row.id, row.weight, row.recordedAt));
        }
        return items;
    }

    @Override
    public long insert(long userId, float weight, long recordedAt) {
        WeightEntryEntity entry = new WeightEntryEntity();
        entry.userId = userId;
        entry.weight = weight;
        entry.recordedAt = recordedAt;
        return roomDao.insert(entry);
    }

    @Override
    public int update(long id, long userId, float weight, long recordedAt) {
        return roomDao.update(id, userId, weight, recordedAt);
    }

    @Override
    public int delete(long id, long userId) {
        return roomDao.delete(id, userId);
    }

    @Override
    public int deleteAll(long userId) {
        return roomDao.deleteAll(userId);
    }

    @Override
    public Float findGoal(long userId) {
        return roomDao.findGoal(userId);
    }

    @Override
    public boolean saveGoal(long userId, float goal) {
        GoalEntity entity = new GoalEntity();
        entity.userId = userId;
        entity.goalWeight = goal;
        return roomDao.upsertGoal(entity) != -1L;
    }

    @Override
    public WeightStats stats(long userId) {
        return orEmpty(roomDao.stats(userId));
    }

    @Override
    public WeightStats statsBetween(long userId, long fromInclusive, long toInclusive) {
        return orEmpty(roomDao.statsBetween(userId, fromInclusive, toInclusive));
    }

    /**
     * An aggregate query over no rows still returns a row, but a caller that never has to
     * check for null is easier to get right than one that does.
     */
    private static WeightStats orEmpty(WeightStats stats) {
        return stats == null ? new WeightStats() : stats;
    }
}
```

</details>

<details markdown="1">
<summary><code>RoomUserDaoAdapter.java</code></summary>

```java
package com.example.cs360project.data.dao;

import com.example.cs360project.data.entity.UserEntity;
import com.example.cs360project.data.migration.LegacyCredential;
import com.example.cs360project.data.model.User;

/**
 * Presents the Room DAO as the {@link UserDao} the repository already expects.
 *
 * The credential crosses this boundary as the single packed string the verifier reads,
 * and is stored as four columns. Splitting on the way in and joining on the way out keeps
 * the schema queryable by algorithm and work factor without the verifier needing to know
 * the columns exist.
 */
public class RoomUserDaoAdapter implements UserDao {

    private final RoomUserDao roomDao;

    public RoomUserDaoAdapter(RoomUserDao roomDao) {
        this.roomDao = roomDao;
    }

    @Override
    public long insert(String username, String encodedPassword) {
        LegacyCredential credential = LegacyCredential.split(encodedPassword);

        UserEntity entity = new UserEntity();
        entity.username = username;
        entity.passwordAlgorithm = credential.getAlgorithm();
        entity.passwordIterations = credential.getIterations();
        entity.passwordSalt = credential.getSalt();
        entity.passwordHash = credential.getHash();

        try {
            return roomDao.insert(entity);
        } catch (RuntimeException e) {
            // The unique index on username rejected it. A duplicate registration is a
            // failure to report, not an exception to propagate into the UI.
            return -1L;
        }
    }

    @Override
    public User findByUsername(String username) {
        UserEntity entity = roomDao.findByUsername(username);
        if (entity == null) {
            return null;
        }

        LegacyCredential credential = LegacyCredential.of(
                entity.passwordAlgorithm, entity.passwordIterations,
                entity.passwordSalt, entity.passwordHash);

        return new User(entity.id, entity.username, credential.toPacked());
    }

    /**
     * Counts credentials stored below a work factor.
     *
     * The iteration count has its own column, so this is one query.
     *
     * @param minimumIterations the work factor to compare against
     * @return how many stored credentials fall below it
     */
    public int countCredentialsBelow(int minimumIterations) {
        return roomDao.countCredentialsBelow(minimumIterations);
    }
}
```

</details>

## Models

<details markdown="1">
<summary><code>WeightItem.java</code></summary>

```java
package com.example.cs360project.data.model;

import java.text.DateFormat;
import java.util.Date;
import java.util.Locale;

/**
 * One logged weight entry.
 *
 * The weight is a float so arithmetic on it needs no parsing, and the timestamp is epoch
 * milliseconds so it sorts, subtracts, and sits in a BETWEEN. Storing the date as display
 * text meant none of those worked: it sorted alphabetically, no two devices in different
 * locales agreed on the format, and no index could help a date range.
 *
 * Display formatting happens here, at the point of display, and uses the device locale on
 * purpose. Locale belongs in what the user reads, not in what the database holds.
 */
public class WeightItem {

    private final long id;
    private final float weight;
    private final long recordedAt;

    public WeightItem(long id, float weight, long recordedAt) {
        this.id = id;
        this.weight = weight;
        this.recordedAt = recordedAt;
    }

    public long getId() {
        return id;
    }

    /** The stored value, ready for arithmetic without parsing. */
    public float getWeight() {
        return weight;
    }

    /** Epoch milliseconds. */
    public long getRecordedAt() {
        return recordedAt;
    }

    /**
     * @return the weight rendered for display, one decimal place
     */
    public String getFormattedWeight() {
        return String.format(Locale.US, "%.1f lbs", weight);
    }

    /**
     * @return the timestamp rendered in the device's own locale and zone
     */
    public String getFormattedDate() {
        return DateFormat.getDateInstance(DateFormat.MEDIUM).format(new Date(recordedAt));
    }
}
```

</details>

<details markdown="1">
<summary><code>User.java</code></summary>

```java
package com.example.cs360project.data.model;

/**
 * A stored account row.
 *
 * This type stays inside the data layer. It carries the encoded credential, so handing it
 * to a ViewModel or an activity would push credential material into the UI. The repository
 * returns {@link AuthenticatedUser} outward instead.
 */
public class User {

    private final long id;
    private final String username;
    private final String encodedPassword;

    public User(long id, String username, String encodedPassword) {
        this.id = id;
        this.username = username;
        this.encodedPassword = encodedPassword;
    }

    public long getId() {
        return id;
    }

    public String getUsername() {
        return username;
    }

    /** The encoded salt and hash as stored. Never displayed and never logged. */
    public String getEncodedPassword() {
        return encodedPassword;
    }
}
```

</details>

<details markdown="1">
<summary><code>AuthenticatedUser.java</code></summary>

```java
package com.example.cs360project.data.model;

/**
 * The identity of a user who has signed in successfully.
 *
 * Carries no credential material. This is what crosses out of the data layer into the
 * ViewModel and the session.
 */
public class AuthenticatedUser {

    private final long id;
    private final String username;

    public AuthenticatedUser(long id, String username) {
        this.id = id;
        this.username = username;
    }

    public long getId() {
        return id;
    }

    public String getUsername() {
        return username;
    }
}
```

</details>

<details markdown="1">
<summary><code>WeightStats.java</code></summary>

```java
package com.example.cs360project.data.model;

import androidx.room.ColumnInfo;

/**
 * Summary figures computed by the database: count, extremes, mean, and the span of the
 * series. One query, one row.
 *
 * The sequential calculations are elsewhere, in the analytics package.
 *
 * Boxed types because every aggregate except COUNT returns null over an empty set.
 */
public class WeightStats {

    @ColumnInfo(name = "entryCount")
    public int entryCount;

    @ColumnInfo(name = "minWeight")
    public Float minWeight;

    @ColumnInfo(name = "maxWeight")
    public Float maxWeight;

    @ColumnInfo(name = "averageWeight")
    public Double averageWeight;

    @ColumnInfo(name = "firstRecordedAt")
    public Long firstRecordedAt;

    @ColumnInfo(name = "lastRecordedAt")
    public Long lastRecordedAt;

    public boolean isEmpty() {
        return entryCount == 0;
    }

    /**
     * @return the span of the series in milliseconds, or 0 when it holds fewer than two entries
     */
    public long spanMillis() {
        if (firstRecordedAt == null || lastRecordedAt == null) {
            return 0L;
        }
        return lastRecordedAt - firstRecordedAt;
    }
}
```

</details>

## Migration

<details markdown="1">
<summary><code>Migrations.java</code></summary>

```java
package com.example.cs360project.data.migration;

import android.content.ContentValues;
import android.database.Cursor;

import androidx.annotation.NonNull;
import androidx.room.migration.Migration;
import androidx.sqlite.db.SupportSQLiteDatabase;

import java.util.ArrayList;
import java.util.List;

/**
 * Upgrade from the first schema to the second without losing anything.
 *
 * Version 1 upgraded by dropping all three tables, so a schema change erased the user's
 * history. This carries the rows across instead.
 *
 * The order matters. The new tables have foreign keys, so users must exist before anything
 * references them, and the legacy rows have no owner, so one is chosen before the weights
 * move. Room wraps the whole thing in a transaction, so a failure leaves the database on
 * version 1 with its data intact.
 *
 * Two steps need Java: splitting the packed credential and parsing the legacy timestamps.
 * Both are pure functions in this package with unit tests. Everything else is plain SQL.
 */
public final class Migrations {

    static final String CREATE_USERS =
            "CREATE TABLE IF NOT EXISTS `users` ("
                    + "`id` INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL, "
                    + "`username` TEXT NOT NULL, "
                    + "`passwordAlgorithm` TEXT NOT NULL, "
                    + "`passwordIterations` INTEGER NOT NULL, "
                    + "`passwordSalt` TEXT NOT NULL, "
                    + "`passwordHash` TEXT NOT NULL)";

    static final String CREATE_USERS_INDEX =
            "CREATE UNIQUE INDEX IF NOT EXISTS `index_users_username` ON `users` (`username`)";

    static final String CREATE_WEIGHT_ENTRIES =
            "CREATE TABLE IF NOT EXISTS `weight_entries` ("
                    + "`id` INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL, "
                    + "`userId` INTEGER NOT NULL, "
                    + "`weight` REAL NOT NULL, "
                    + "`recordedAt` INTEGER NOT NULL, "
                    + "FOREIGN KEY(`userId`) REFERENCES `users`(`id`) "
                    + "ON UPDATE NO ACTION ON DELETE CASCADE )";

    static final String CREATE_WEIGHT_ENTRIES_INDEX =
            "CREATE INDEX IF NOT EXISTS `index_weight_entries_userId_recordedAt` "
                    + "ON `weight_entries` (`userId`, `recordedAt`)";

    static final String CREATE_GOALS =
            "CREATE TABLE IF NOT EXISTS `goals` ("
                    + "`id` INTEGER PRIMARY KEY AUTOINCREMENT NOT NULL, "
                    + "`userId` INTEGER NOT NULL, "
                    + "`goalWeight` REAL NOT NULL, "
                    + "FOREIGN KEY(`userId`) REFERENCES `users`(`id`) "
                    + "ON UPDATE NO ACTION ON DELETE CASCADE )";

    static final String CREATE_GOALS_INDEX =
            "CREATE UNIQUE INDEX IF NOT EXISTS `index_goals_userId` ON `goals` (`userId`)";

    /** Credential columns are filled in afterwards, once the packed value has been split. */
    static final String COPY_USERS =
            "INSERT INTO `users` "
                    + "(`id`, `username`, `passwordAlgorithm`, `passwordIterations`, "
                    + "`passwordSalt`, `passwordHash`) "
                    + "SELECT `_id`, `username`, 'unmigrated', 0, '', `password` FROM `login`";

    /** recordedAt is filled in afterwards, once the legacy text has been parsed. */
    static final String COPY_WEIGHTS =
            "INSERT INTO `weight_entries` (`id`, `userId`, `weight`, `recordedAt`) "
                    + "SELECT `_id`, ?, `weight`, 0 FROM `weights`";

    static final String COPY_GOAL =
            "INSERT INTO `goals` (`userId`, `goalWeight`) "
                    + "SELECT ?, `goal_weight` FROM `goal` LIMIT 1";

    /**
     * Account that adopts rows left behind when no login exists.
     *
     * Version 1 let data be logged without registering, so some installs have rows that
     * belong to nobody. They are attached here. No password verifies against this account.
     */
    static final String CREATE_ORPHAN_OWNER =
            "INSERT INTO `users` "
                    + "(`username`, `passwordAlgorithm`, `passwordIterations`, "
                    + "`passwordSalt`, `passwordHash`) "
                    + "VALUES ('migrated-data', 'unmigrated', 0, '', '')";

    static final String DROP_LOGIN = "DROP TABLE IF EXISTS `login`";
    static final String DROP_WEIGHTS = "DROP TABLE IF EXISTS `weights`";
    static final String DROP_GOAL = "DROP TABLE IF EXISTS `goal`";

    private Migrations() {
    }

    public static final Migration MIGRATION_1_2 = new Migration(1, 2) {
        @Override
        public void migrate(@NonNull SupportSQLiteDatabase database) {
            database.execSQL(CREATE_USERS);
            database.execSQL(CREATE_USERS_INDEX);
            database.execSQL(CREATE_WEIGHT_ENTRIES);
            database.execSQL(CREATE_WEIGHT_ENTRIES_INDEX);
            database.execSQL(CREATE_GOALS);
            database.execSQL(CREATE_GOALS_INDEX);

            database.execSQL(COPY_USERS);
            splitStoredCredentials(database);

            long ownerId = resolveOwnerId(database);
            database.execSQL(COPY_WEIGHTS, new Object[]{ownerId});
            database.execSQL(COPY_GOAL, new Object[]{ownerId});
            convertTimestamps(database);

            database.execSQL(DROP_LOGIN);
            database.execSQL(DROP_WEIGHTS);
            database.execSQL(DROP_GOAL);
        }
    };

    /**
     * Fills the credential columns by splitting the value copied from the old table.
     */
    private static void splitStoredCredentials(SupportSQLiteDatabase database) {
        try (Cursor cursor = database.query("SELECT `id`, `passwordHash` FROM `users`")) {
            while (cursor.moveToNext()) {
                long id = cursor.getLong(0);
                LegacyCredential credential = LegacyCredential.split(cursor.getString(1));

                ContentValues values = new ContentValues();
                values.put("passwordAlgorithm", credential.getAlgorithm());
                values.put("passwordIterations", credential.getIterations());
                values.put("passwordSalt", credential.getSalt());
                values.put("passwordHash", credential.getHash());

                database.update("users", 0, values, "`id` = ?", new Object[]{id});
            }
        }
    }

    /**
     * Converts every copied timestamp from display text to epoch milliseconds.
     *
     * Read in primary key order, parsed as a batch, then written back. Converting one
     * row at a time has nothing to interpolate between, so an unreadable date could not
     * be filled from the rows either side of it.
     */
    private static void convertTimestamps(SupportSQLiteDatabase database) {
        List<Long> ids = new ArrayList<>();
        List<Long> parsed = new ArrayList<>();

        try (Cursor cursor = database.query(
                "SELECT `weight_entries`.`id`, `weights`.`date` "
                        + "FROM `weight_entries` "
                        + "JOIN `weights` ON `weights`.`_id` = `weight_entries`.`id` "
                        + "ORDER BY `weight_entries`.`id`")) {
            while (cursor.moveToNext()) {
                ids.add(cursor.getLong(0));
                parsed.add(LegacyTimestampParser.parse(cursor.getString(1)));
            }
        }

        if (ids.isEmpty()) {
            return;
        }

        long[] resolved = LegacyTimestampParser.resolveSequence(
                parsed.toArray(new Long[0]), System.currentTimeMillis());

        for (int i = 0; i < ids.size(); i++) {
            ContentValues values = new ContentValues();
            values.put("recordedAt", resolved[i]);
            database.update("weight_entries", 0, values,
                    "`id` = ?", new Object[]{ids.get(i)});
        }
    }

    /**
     * Picks the account the ownerless legacy rows are attached to.
     *
     * The lowest account id, which on a single user install is the only one. When there is
     * no account at all, one is created to hold the data.
     */
    private static long resolveOwnerId(SupportSQLiteDatabase database) {
        try (Cursor cursor = database.query("SELECT MIN(`id`) FROM `users`")) {
            if (cursor.moveToFirst() && !cursor.isNull(0)) {
                return cursor.getLong(0);
            }
        }

        database.execSQL(CREATE_ORPHAN_OWNER);
        try (Cursor cursor = database.query("SELECT MIN(`id`) FROM `users`")) {
            cursor.moveToFirst();
            return cursor.getLong(0);
        }
    }
}
```

</details>

<details markdown="1">
<summary><code>LegacyTimestampParser.java</code></summary>

```java
package com.example.cs360project.data.migration;

import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Locale;
import java.util.TimeZone;

/**
 * Reads the timestamps the first version of the application wrote.
 *
 * Those rows hold the output of Date.toString, which looks like
 * "Mon Sep 07 14:23:11 MDT 2026". Month and day names come from the device locale and the
 * zone is an ambiguous abbreviation, so reading them back is best effort.
 *
 * Tries US English first, then the device locale. A string neither can read returns null
 * and the caller decides what to do.
 *
 * No Android imports, so the parsing rules are covered by JVM unit tests.
 */
public final class LegacyTimestampParser {

    private static final String LEGACY_PATTERN = "EEE MMM dd HH:mm:ss zzz yyyy";

    private LegacyTimestampParser() {
    }

    /**
     * Converts one legacy timestamp string to epoch milliseconds.
     *
     * @param legacy the stored text, may be null or empty
     * @return epoch milliseconds, or null when the text cannot be read
     */
    public static Long parse(String legacy) {
        if (legacy == null || legacy.trim().isEmpty()) {
            return null;
        }
        String text = legacy.trim();

        Long parsed = tryParse(text, Locale.US);
        if (parsed != null) {
            return parsed;
        }
        Locale deviceLocale = Locale.getDefault();
        if (!Locale.US.equals(deviceLocale)) {
            parsed = tryParse(text, deviceLocale);
        }
        return parsed;
    }

    /**
     * Converts a legacy timestamp, falling back when it cannot be read.
     *
     * The fallback keeps the row. A weight with an unreadable date is still a weight.
     *
     * @param legacy   the stored text
     * @param fallback epoch milliseconds to use when parsing fails
     * @return epoch milliseconds, never null
     */
    public static long parseOrFallback(String legacy, long fallback) {
        Long parsed = parse(legacy);
        return parsed == null ? fallback : parsed;
    }

    private static Long tryParse(String text, Locale locale) {
        try {
            SimpleDateFormat format = new SimpleDateFormat(LEGACY_PATTERN, locale);
            format.setLenient(false);
            return format.parse(text).getTime();
        } catch (ParseException | IllegalArgumentException e) {
            return null;
        }
    }

    /**
     * Turns a run of parse results into strictly increasing timestamps.
     *
     * An unreadable row still has a position, since the primary key says which entry came
     * first. Each gap is filled from its neighbours so the row keeps that position.
     *
     * The result is strictly increasing even where the stored dates were not. Downstream
     * calculations read the series in order, and two entries sharing an instant have no
     * defined order.
     *
     * @param parsed        one entry per row in primary key order, null where unreadable
     * @param fallbackStart epoch milliseconds to start from when nothing could be read
     * @return epoch milliseconds, same length as the input, strictly increasing
     */
    public static long[] resolveSequence(Long[] parsed, long fallbackStart) {
        if (parsed == null || parsed.length == 0) {
            return new long[0];
        }

        long[] resolved = new long[parsed.length];
        boolean anyAssigned = false;
        long previous = 0L;

        for (int i = 0; i < parsed.length; i++) {
            if (parsed[i] != null) {
                long candidate = parsed[i];
                if (anyAssigned && candidate <= previous) {
                    candidate = previous + 1L;
                }
                resolved[i] = candidate;
            } else if (anyAssigned) {
                resolved[i] = previous + 1L;
            } else {
                // Leading gap. Anchor below the first readable date so these rows keep
                // their position at the front rather than jumping ahead of it.
                Long nextKnown = nextNonNull(parsed, i);
                resolved[i] = nextKnown == null
                        ? fallbackStart + i
                        : nextKnown - countNullsUntilNext(parsed, i);
            }
            previous = resolved[i];
            anyAssigned = true;
        }
        return resolved;
    }

    private static Long nextNonNull(Long[] values, int from) {
        for (int i = from; i < values.length; i++) {
            if (values[i] != null) {
                return values[i];
            }
        }
        return null;
    }

    private static long countNullsUntilNext(Long[] values, int from) {
        long count = 0L;
        for (int i = from; i < values.length && values[i] == null; i++) {
            count++;
        }
        return count;
    }

    /**
     * @return the format string the first version wrote, for tests and documentation
     */
    public static String legacyPattern() {
        return LEGACY_PATTERN;
    }

    /**
     * @return UTC, the zone synthetic timestamps are anchored in
     */
    static TimeZone syntheticZone() {
        return TimeZone.getTimeZone("UTC");
    }
}
```

</details>

<details markdown="1">
<summary><code>LegacyCredential.java</code></summary>

```java
package com.example.cs360project.data.migration;

/**
 * Splits a packed credential string into the columns the new schema stores it in.
 *
 * The first version kept the whole credential in one TEXT column as
 * algorithm$iterations$salt$hash, because adding a column meant a schema change and the
 * upgrade path at the time destroyed the table. With a real migration available the four
 * parts get columns of their own.
 *
 * Anything that does not split into four recognizable parts is reported as unparsed. The
 * earliest version stored passwords as plaintext, so a value not in the packed format is
 * probably one of those, and it is marked rather than written into a column named
 * passwordHash.
 */
public final class LegacyCredential {

    private final String algorithm;
    private final int iterations;
    private final String salt;
    private final String hash;
    private final boolean parsed;

    private LegacyCredential(String algorithm, int iterations, String salt,
                             String hash, boolean parsed) {
        this.algorithm = algorithm;
        this.iterations = iterations;
        this.salt = salt;
        this.hash = hash;
        this.parsed = parsed;
    }

    /**
     * Builds a credential from columns already split.
     *
     * @param algorithm  key derivation function identifier
     * @param iterations work factor
     * @param salt       base64 salt
     * @param hash       base64 derived key
     * @return the credential, marked unparsed when the parts are not usable
     */
    public static LegacyCredential of(String algorithm, int iterations,
                                      String salt, String hash) {
        boolean usable = algorithm != null && !algorithm.isEmpty()
                && !"unmigrated".equals(algorithm)
                && iterations > 0
                && salt != null && !salt.isEmpty()
                && hash != null && !hash.isEmpty();

        if (!usable) {
            return unparsed(hash == null ? "" : hash);
        }
        return new LegacyCredential(algorithm, iterations, salt, hash, true);
    }

    /**
     * Splits a stored credential.
     *
     * @param stored the value from the old password column, may be null
     * @return the parts, or an unparsed result
     */
    public static LegacyCredential split(String stored) {
        if (stored == null) {
            return unparsed("");
        }
        String[] parts = stored.split("\\$");
        if (parts.length != 4) {
            return unparsed(stored);
        }
        int iterations;
        try {
            iterations = Integer.parseInt(parts[1]);
        } catch (NumberFormatException e) {
            return unparsed(stored);
        }
        if (iterations <= 0 || parts[0].isEmpty() || parts[2].isEmpty() || parts[3].isEmpty()) {
            return unparsed(stored);
        }
        return new LegacyCredential(parts[0], iterations, parts[2], parts[3], true);
    }

    /**
     * Marks a value that could not be split.
     *
     * The algorithm column records that the row needs attention and the stored text is
     * kept in the hash column so nothing is thrown away. No password verifies against it,
     * because the verifier only accepts a recognized algorithm, so the account is locked
     * rather than silently accepting anything.
     */
    private static LegacyCredential unparsed(String original) {
        return new LegacyCredential("unmigrated", 0, "", original, false);
    }

    public String getAlgorithm() {
        return algorithm;
    }

    public int getIterations() {
        return iterations;
    }

    public String getSalt() {
        return salt;
    }

    public String getHash() {
        return hash;
    }

    /** False when the stored value was not in the packed format. */
    public boolean isParsed() {
        return parsed;
    }

    /**
     * Rebuilds the packed form the verifier reads.
     *
     * @return the four parts joined, or null when the credential was never parsed
     */
    public String toPacked() {
        if (!parsed) {
            return null;
        }
        return algorithm + "$" + iterations + "$" + salt + "$" + hash;
    }
}
```

</details>

## Security

<details markdown="1">
<summary><code>PasswordHasher.java</code></summary>

```java
package com.example.cs360project.security;

import java.nio.charset.StandardCharsets;
import java.security.GeneralSecurityException;
import java.security.MessageDigest;
import java.security.SecureRandom;
import java.util.Arrays;
import java.util.Base64;

import javax.crypto.SecretKeyFactory;
import javax.crypto.spec.PBEKeySpec;

/**
 * Salted, iterated password hashing.
 *
 * PBKDF2 with HMAC-SHA256, a random salt per password, and a high iteration count. The
 * salt stops two identical passwords storing the same value and makes precomputed tables
 * useless. The iteration count sets the cost of each guess against a stolen file.
 *
 * Encoded as pbkdf2_sha256$iterations$base64-salt$base64-hash. The iteration count
 * travels with the value, so the cost can be raised without invalidating stored
 * credentials.
 *
 * No Android imports. PBKDF2 and {@link Base64} are available from API 26 and minSdk is 34.
 */
public final class PasswordHasher {

    private static final String ALGORITHM = "PBKDF2WithHmacSHA256";
    private static final String FORMAT_ID = "pbkdf2_sha256";
    private static final String SEPARATOR = "\\$";
    private static final char SEPARATOR_CHAR = '$';

    private static final int SALT_LENGTH_BYTES = 16;
    private static final int KEY_LENGTH_BITS = 256;
    private static final int DEFAULT_ITERATIONS = 120_000;
    private static final int EXPECTED_FIELDS = 4;

    private static final SecureRandom SECURE_RANDOM = new SecureRandom();

    private PasswordHasher() {
        // Utility class, not instantiable.
    }

    /**
     * Hashes a password for storage.
     *
     * The caller owns the array and should clear it once this method returns.
     *
     * @param password the password characters
     * @return the encoded salt and hash, safe to store
     * @throws IllegalStateException if the platform lacks the required algorithm
     */
    public static String hash(char[] password) {
        byte[] salt = new byte[SALT_LENGTH_BYTES];
        SECURE_RANDOM.nextBytes(salt);

        byte[] derived = derive(password, salt, DEFAULT_ITERATIONS);
        try {
            Base64.Encoder encoder = Base64.getEncoder();
            return FORMAT_ID
                    + SEPARATOR_CHAR + DEFAULT_ITERATIONS
                    + SEPARATOR_CHAR + encoder.encodeToString(salt)
                    + SEPARATOR_CHAR + encoder.encodeToString(derived);
        } finally {
            Arrays.fill(derived, (byte) 0);
        }
    }

    /**
     * Verifies a password against a stored encoded value.
     *
     * Compares with {@link MessageDigest#isEqual(byte[], byte[])}, which does not return
     * early on the first differing byte and so does not leak how much of a guess was right.
     *
     * A malformed stored value returns false. It is never treated as a match.
     *
     * @param password the password characters to check
     * @param encoded  the stored value produced by {@link #hash(char[])}
     * @return true when the password matches
     */
    public static boolean verify(char[] password, String encoded) {
        if (password == null || encoded == null) {
            return false;
        }

        String[] fields = encoded.split(SEPARATOR);
        if (fields.length != EXPECTED_FIELDS || !FORMAT_ID.equals(fields[0])) {
            return false;
        }

        int iterations;
        byte[] salt;
        byte[] expected;
        try {
            iterations = Integer.parseInt(fields[1]);
            Base64.Decoder decoder = Base64.getDecoder();
            salt = decoder.decode(fields[2]);
            expected = decoder.decode(fields[3]);
        } catch (IllegalArgumentException e) {
            // Covers both a non-numeric iteration count and malformed Base64, since
            // NumberFormatException extends IllegalArgumentException.
            return false;
        }

        if (iterations <= 0 || salt.length == 0 || expected.length == 0) {
            return false;
        }

        byte[] actual = derive(password, salt, iterations);
        try {
            return MessageDigest.isEqual(expected, actual);
        } finally {
            Arrays.fill(actual, (byte) 0);
        }
    }

    /**
     * Produces a stored value that no password matches.
     *
     * Used when a sign in names an account that does not exist. Verifying against this
     * value costs the same as verifying against a real one, so the response time does not
     * reveal whether the username is registered.
     *
     * @return an encoded value with a random salt and unreachable hash
     */
    public static String unmatchableHash() {
        byte[] salt = new byte[SALT_LENGTH_BYTES];
        byte[] filler = new byte[KEY_LENGTH_BITS / Byte.SIZE];
        SECURE_RANDOM.nextBytes(salt);
        SECURE_RANDOM.nextBytes(filler);

        Base64.Encoder encoder = Base64.getEncoder();
        return FORMAT_ID
                + SEPARATOR_CHAR + DEFAULT_ITERATIONS
                + SEPARATOR_CHAR + encoder.encodeToString(salt)
                + SEPARATOR_CHAR + encoder.encodeToString(filler);
    }

    /**
     * Runs the key derivation function.
     *
     * @param password   password characters
     * @param salt       per-password random salt
     * @param iterations work factor
     * @return the derived key bytes
     * @throws IllegalStateException if the platform lacks the required algorithm
     */
    private static byte[] derive(char[] password, byte[] salt, int iterations) {
        PBEKeySpec spec = new PBEKeySpec(password, salt, iterations, KEY_LENGTH_BITS);
        try {
            SecretKeyFactory factory = SecretKeyFactory.getInstance(ALGORITHM);
            return factory.generateSecret(spec).getEncoded();
        } catch (GeneralSecurityException e) {
            // Every supported platform provides PBKDF2WithHmacSHA256. If it is missing,
            // failing loudly is correct. Silently falling back to a weaker scheme is how
            // credential storage quietly degrades.
            throw new IllegalStateException("PBKDF2 is unavailable on this platform", e);
        } finally {
            spec.clearPassword();
        }
    }

    /**
     * Clears a character array in place.
     *
     * @param chars array to overwrite, may be null
     */
    public static void wipe(char[] chars) {
        if (chars != null) {
            Arrays.fill(chars, '\0');
        }
    }

    /**
     * Copies a CharSequence into a character array the caller can wipe.
     *
     * Reading an EditText through {@code getText().toString()} creates an immutable String
     * that cannot be cleared. Copying the characters directly avoids that.
     *
     * @param sequence source text, may be null
     * @return a new array holding the characters, never null
     */
    public static char[] toCharArray(CharSequence sequence) {
        if (sequence == null) {
            return new char[0];
        }
        char[] chars = new char[sequence.length()];
        for (int i = 0; i < sequence.length(); i++) {
            chars[i] = sequence.charAt(i);
        }
        return chars;
    }

    /**
     * Exposes the charset used for encoding, for tests and documentation.
     *
     * @return the charset PBKDF2 input is interpreted in by the platform
     */
    static String charsetName() {
        return StandardCharsets.UTF_8.name();
    }
}
```

</details>

<details markdown="1">
<summary><code>SessionManager.java</code></summary>

```java
package com.example.cs360project.security;

/**
 * Holds the identity of the signed in user for the lifetime of the process.
 *
 * Carries the identity that queries are scoped by. Without it a sign in confirms a row
 * matched and nothing downstream knows who is using the app.
 *
 * In memory only, so the session ends when the process does and the user signs in again.
 */
public final class SessionManager {

    /** Sentinel returned by {@link #getUserId()} when no session is active. */
    public static final long NO_USER = -1L;

    private static final SessionManager INSTANCE = new SessionManager();

    private long userId = NO_USER;
    private String username;

    private SessionManager() {
        // Single instance, obtained through getInstance.
    }

    public static SessionManager getInstance() {
        return INSTANCE;
    }

    /**
     * Records a successful sign in.
     *
     * @param userId   primary key of the authenticated row
     * @param username the account name, for display
     */
    public synchronized void startSession(long userId, String username) {
        this.userId = userId;
        this.username = username;
    }

    /** Clears the session. */
    public synchronized void endSession() {
        this.userId = NO_USER;
        this.username = null;
    }

    public synchronized boolean isActive() {
        return userId != NO_USER;
    }

    /**
     * @return the signed in user id, or {@link #NO_USER} when no session is active
     */
    public synchronized long getUserId() {
        return userId;
    }

    /**
     * @return the signed in username, or null when no session is active
     */
    public synchronized String getUsername() {
        return username;
    }
}
```

</details>

## Utilities

<details markdown="1">
<summary><code>Validator.java</code></summary>

```java
package com.example.cs360project.util;

/**
 * Single source of truth for input validation rules.
 *
 * Every rule lives here, so two screens cannot disagree about what a valid weight is.
 * Parsing happens inside the check and reports a message, so a non-numeric entry never
 * reaches an unguarded {@link NumberFormatException}.
 *
 * Final with a private constructor because it holds no state. No Android imports, so the
 * rules can be tested on the JVM.
 */
public final class Validator {

    /** Lowest weight accepted, in pounds. Below this the entry is almost certainly a typo. */
    public static final float MIN_WEIGHT_LBS = 20f;

    /** Highest weight accepted, in pounds. Above the heaviest recorded human weight. */
    public static final float MAX_WEIGHT_LBS = 1500f;

    public static final int MIN_USERNAME_LENGTH = 3;
    public static final int MAX_USERNAME_LENGTH = 32;
    public static final int MIN_PASSWORD_LENGTH = 8;
    public static final int MAX_PASSWORD_LENGTH = 128;

    private Validator() {
        // Utility class, not instantiable.
    }

    /**
     * Validates a username.
     *
     * @param raw text straight from the input field, possibly null
     * @return a valid result, or an invalid result carrying the reason
     */
    public static ValidationResult validateUsername(String raw) {
        String username = raw == null ? "" : raw.trim();

        if (username.isEmpty()) {
            return ValidationResult.invalid("Enter a username");
        }
        if (username.length() < MIN_USERNAME_LENGTH) {
            return ValidationResult.invalid(
                    "Username must be at least " + MIN_USERNAME_LENGTH + " characters");
        }
        if (username.length() > MAX_USERNAME_LENGTH) {
            return ValidationResult.invalid(
                    "Username must be " + MAX_USERNAME_LENGTH + " characters or fewer");
        }
        for (int i = 0; i < username.length(); i++) {
            char c = username.charAt(i);
            boolean allowed = Character.isLetterOrDigit(c) || c == '_' || c == '.' || c == '-';
            if (!allowed) {
                return ValidationResult.invalid(
                        "Username may contain only letters, digits, and _ . -");
            }
        }
        return ValidationResult.valid();
    }

    /**
     * Validates a password at registration time.
     *
     * Takes a character array so the caller can clear it once hashed. A String cannot be
     * cleared and stays on the heap until collected.
     *
     * @param raw characters straight from the input field, possibly null
     * @return a valid result, or an invalid result carrying the reason
     */
    public static ValidationResult validateNewPassword(char[] raw) {
        int length = raw == null ? 0 : raw.length;

        if (length == 0) {
            return ValidationResult.invalid("Enter a password");
        }
        if (length < MIN_PASSWORD_LENGTH) {
            return ValidationResult.invalid(
                    "Password must be at least " + MIN_PASSWORD_LENGTH + " characters");
        }
        if (length > MAX_PASSWORD_LENGTH) {
            return ValidationResult.invalid(
                    "Password must be " + MAX_PASSWORD_LENGTH + " characters or fewer");
        }
        return ValidationResult.valid();
    }

    /**
     * Validates a password supplied at sign in.
     *
     * This checks only that something was entered. Length and composition rules are not
     * applied here, because an existing account may predate the current rules and because
     * rejecting a sign in attempt on format tells an attacker something about the stored
     * password.
     *
     * @param raw characters straight from the input field, possibly null
     * @return a valid result, or an invalid result carrying the reason
     */
    public static ValidationResult validateSubmittedPassword(char[] raw) {
        if (raw == null || raw.length == 0) {
            return ValidationResult.invalid("Enter a password");
        }
        return ValidationResult.valid();
    }

    /**
     * Validates a required weight entry and parses it.
     *
     * @param raw text straight from the input field, possibly null
     * @return a valid result carrying the parsed weight, or an invalid result
     */
    public static ValidationResult validateWeight(String raw) {
        String text = raw == null ? "" : raw.trim();

        if (text.isEmpty()) {
            return ValidationResult.invalid("Enter a weight");
        }
        return parseInRange(text, "Weight");
    }

    /**
     * Validates an optional goal weight entry.
     *
     * A blank entry is valid and carries no value, which is how the caller knows to leave
     * the stored goal untouched.
     *
     * @param raw text straight from the input field, possibly null
     * @return a valid result with or without a value, or an invalid result
     */
    public static ValidationResult validateOptionalGoal(String raw) {
        String text = raw == null ? "" : raw.trim();

        if (text.isEmpty()) {
            return ValidationResult.valid();
        }
        return parseInRange(text, "Goal weight");
    }

    /**
     * Parses a non-empty numeric string and range checks it.
     *
     * @param text  trimmed, non-empty input
     * @param label field name used in any failure message
     * @return a valid result carrying the parsed value, or an invalid result
     */
    private static ValidationResult parseInRange(String text, String label) {
        float parsed;
        try {
            parsed = Float.parseFloat(text);
        } catch (NumberFormatException e) {
            return ValidationResult.invalid(label + " must be a number");
        }

        if (Float.isNaN(parsed) || Float.isInfinite(parsed)) {
            return ValidationResult.invalid(label + " must be a number");
        }
        if (parsed < MIN_WEIGHT_LBS || parsed > MAX_WEIGHT_LBS) {
            return ValidationResult.invalid(label + " must be between "
                    + (int) MIN_WEIGHT_LBS + " and " + (int) MAX_WEIGHT_LBS + " lbs");
        }
        return ValidationResult.valid(parsed);
    }
}
```

</details>

<details markdown="1">
<summary><code>ValidationResult.java</code></summary>

```java
package com.example.cs360project.util;

/**
 * Outcome of a single validation check.
 *
 * Valid or invalid, with a user facing message when invalid. Numeric checks also carry
 * the parsed value, so the caller never parses the input a second time.
 *
 * No Android imports, so validation rules are covered by JVM unit tests.
 */
public final class ValidationResult {

    private final boolean valid;
    private final String message;
    private final float value;
    private final boolean hasValue;

    private ValidationResult(boolean valid, String message, float value, boolean hasValue) {
        this.valid = valid;
        this.message = message;
        this.value = value;
        this.hasValue = hasValue;
    }

    /** A passing result that carries no parsed value. */
    public static ValidationResult valid() {
        return new ValidationResult(true, null, 0f, false);
    }

    /** A passing result that carries a parsed numeric value. */
    public static ValidationResult valid(float value) {
        return new ValidationResult(true, null, value, true);
    }

    /** A failing result carrying a user facing message. */
    public static ValidationResult invalid(String message) {
        return new ValidationResult(false, message, 0f, false);
    }

    public boolean isValid() {
        return valid;
    }

    /** The failure message, or null when the result is valid. */
    public String getMessage() {
        return message;
    }

    /** True when a parsed numeric value is present. Optional fields left blank return false. */
    public boolean hasValue() {
        return hasValue;
    }

    /**
     * The parsed numeric value.
     *
     * @throws IllegalStateException if no value is present. Callers check {@link #hasValue()} first.
     */
    public float getValue() {
        if (!hasValue) {
            throw new IllegalStateException("No parsed value is present on this result");
        }
        return value;
    }
}
```

</details>

<details markdown="1">
<summary><code>AppExecutors.java</code></summary>

```java
package com.example.cs360project.util;

import android.os.Handler;
import android.os.Looper;

import java.util.concurrent.Executor;
import java.util.concurrent.Executors;

/**
 * Thread pools for the application.
 *
 * Persistence runs on {@link #diskIO()} and results come back on {@link #mainThread()}.
 * A query on the main thread blocks the UI and risks an Application Not Responding dialog.
 *
 * One background thread, not a pool. SQLite serializes writes anyway, and one thread gives
 * the repository ordered execution.
 */
public final class AppExecutors {

    private static volatile AppExecutors instance;

    private final Executor diskIO;
    private final Executor mainThread;

    private AppExecutors() {
        this(Executors.newSingleThreadExecutor(), new MainThreadExecutor());
    }

    /**
     * Builds an instance with supplied executors.
     *
     * Exists so a unit test can pass {@link Runnable#run()} style executors and drive the
     * repository layer synchronously on the JVM. Production code uses
     * {@link #getInstance()}.
     *
     * @param diskIO     executor for blocking work
     * @param mainThread executor that results are delivered on
     */
    public AppExecutors(Executor diskIO, Executor mainThread) {
        this.diskIO = diskIO;
        this.mainThread = mainThread;
    }

    public static AppExecutors getInstance() {
        if (instance == null) {
            synchronized (AppExecutors.class) {
                if (instance == null) {
                    instance = new AppExecutors();
                }
            }
        }
        return instance;
    }

    /** Executor for database and other blocking work. */
    public Executor diskIO() {
        return diskIO;
    }

    /** Executor that posts to the Android main looper. */
    public Executor mainThread() {
        return mainThread;
    }

    /** Posts work to the main looper so results can touch views safely. */
    private static final class MainThreadExecutor implements Executor {
        private final Handler handler = new Handler(Looper.getMainLooper());

        @Override
        public void execute(Runnable command) {
            handler.post(command);
        }
    }
}
```

</details>

<details markdown="1">
<summary><code>Callback.java</code></summary>

```java
package com.example.cs360project.util;

/**
 * Single result callback used by the repository layer.
 *
 * Repository methods work on a background executor and call back on the main thread.
 *
 * @param <T> type of the value delivered to the caller
 */
public interface Callback<T> {

    /**
     * Invoked on the main thread once the work has finished.
     *
     * @param result the outcome of the operation
     */
    void onResult(T result);
}
```

</details>

<details markdown="1">
<summary><code>Outcome.java</code></summary>

```java
package com.example.cs360project.util;

/**
 * Success or failure of an operation, with a reason when it failed.
 *
 * Repository methods return this so a caller can tell a saved row from a lost one.
 *
 * @param <T> type of the value carried on success
 */
public final class Outcome<T> {

    private final boolean success;
    private final T value;
    private final String message;

    private Outcome(boolean success, T value, String message) {
        this.success = success;
        this.value = value;
        this.message = message;
    }

    /**
     * @param value the result, may be null when the operation has no return value
     */
    public static <T> Outcome<T> success(T value) {
        return new Outcome<>(true, value, null);
    }

    /**
     * @param message a user facing reason for the failure
     */
    public static <T> Outcome<T> failure(String message) {
        return new Outcome<>(false, null, message);
    }

    public boolean isSuccess() {
        return success;
    }

    /** The carried value, or null on failure or when the operation returns nothing. */
    public T getValue() {
        return value;
    }

    /** The failure reason, or null on success. */
    public String getMessage() {
        return message;
    }
}
```

</details>

## Tests

<details markdown="1">
<summary><code>ValidatorTest.java</code></summary>

```java
package com.example.cs360project.util;

import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertFalse;
import static org.junit.Assert.assertNotNull;
import static org.junit.Assert.assertTrue;

import org.junit.Test;

/**
 * Tests for the centralized validation rules.
 *
 * These run on the JVM with no device and no emulator. Validation rules inline in a
 * click listener cannot be reached by a test at all, so they live in a class of their own.
 */
public class ValidatorTest {

    @Test
    public void weightRejectsNonNumericInsteadOfThrowing() {
        // Unguarded, Float.parseFloat throws on this and takes the activity down.
        ValidationResult result = Validator.validateWeight("abc");

        assertFalse(result.isValid());
        assertNotNull(result.getMessage());
    }

    @Test
    public void weightRejectsEmpty() {
        assertFalse(Validator.validateWeight("").isValid());
        assertFalse(Validator.validateWeight("   ").isValid());
        assertFalse(Validator.validateWeight(null).isValid());
    }

    @Test
    public void weightAcceptsPlausibleValueAndParsesIt() {
        ValidationResult result = Validator.validateWeight("185.4");

        assertTrue(result.isValid());
        assertTrue(result.hasValue());
        assertEquals(185.4f, result.getValue(), 0.0001f);
    }

    @Test
    public void weightTrimsSurroundingSpace() {
        ValidationResult result = Validator.validateWeight("  185  ");

        assertTrue(result.isValid());
        assertEquals(185f, result.getValue(), 0.0001f);
    }

    @Test
    public void weightRejectsValuesOutsideThePlausibleRange() {
        assertFalse(Validator.validateWeight("0").isValid());
        assertFalse(Validator.validateWeight("-150").isValid());
        assertFalse(Validator.validateWeight("99999").isValid());
    }

    @Test
    public void weightAcceptsTheRangeBoundaries() {
        assertTrue(Validator.validateWeight(
                String.valueOf(Validator.MIN_WEIGHT_LBS)).isValid());
        assertTrue(Validator.validateWeight(
                String.valueOf(Validator.MAX_WEIGHT_LBS)).isValid());
    }

    @Test
    public void weightRejectsNotANumberAndInfinity() {
        assertFalse(Validator.validateWeight("NaN").isValid());
        assertFalse(Validator.validateWeight("Infinity").isValid());
    }

    @Test
    public void optionalGoalTreatsBlankAsValidWithNoValue() {
        ValidationResult result = Validator.validateOptionalGoal("");

        assertTrue(result.isValid());
        assertFalse(result.hasValue());
    }

    @Test
    public void optionalGoalStillRangeChecksWhenSupplied() {
        assertFalse(Validator.validateOptionalGoal("4").isValid());

        ValidationResult result = Validator.validateOptionalGoal("160");
        assertTrue(result.isValid());
        assertTrue(result.hasValue());
        assertEquals(160f, result.getValue(), 0.0001f);
    }

    @Test
    public void usernameRejectsEmptyShortAndOverlong() {
        assertFalse(Validator.validateUsername("").isValid());
        assertFalse(Validator.validateUsername(null).isValid());
        assertFalse(Validator.validateUsername("ab").isValid());

        StringBuilder tooLong = new StringBuilder();
        for (int i = 0; i <= Validator.MAX_USERNAME_LENGTH; i++) {
            tooLong.append('a');
        }
        assertFalse(Validator.validateUsername(tooLong.toString()).isValid());
    }

    @Test
    public void usernameRejectsDisallowedCharacters() {
        assertFalse(Validator.validateUsername("with space").isValid());
        assertFalse(Validator.validateUsername("semi;colon").isValid());
        assertFalse(Validator.validateUsername("quote'mark").isValid());
    }

    @Test
    public void usernameAcceptsLettersDigitsAndTheAllowedPunctuation() {
        assertTrue(Validator.validateUsername("david_noh").isValid());
        assertTrue(Validator.validateUsername("d.noh-1").isValid());
    }

    @Test
    public void newPasswordEnforcesMinimumLength() {
        assertFalse(Validator.validateNewPassword(new char[0]).isValid());
        assertFalse(Validator.validateNewPassword(null).isValid());
        assertFalse(Validator.validateNewPassword("short".toCharArray()).isValid());
        assertTrue(Validator.validateNewPassword("longenough1".toCharArray()).isValid());
    }

    @Test
    public void submittedPasswordChecksOnlyThatSomethingWasEntered() {
        // Length rules are not applied at sign in, so an older account is not locked out
        // and a rejection does not reveal anything about the stored password.
        assertFalse(Validator.validateSubmittedPassword(new char[0]).isValid());
        assertTrue(Validator.validateSubmittedPassword("x".toCharArray()).isValid());
    }
}
```

</details>

<details markdown="1">
<summary><code>PasswordHasherTest.java</code></summary>

```java
package com.example.cs360project.security;

import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertFalse;
import static org.junit.Assert.assertNotEquals;
import static org.junit.Assert.assertTrue;

import org.junit.Test;

/**
 * Tests for password hashing and verification.
 *
 * PBKDF2 and Base64 come from the Java platform, so these run on the JVM.
 */
public class PasswordHasherTest {

    @Test
    public void verifyAcceptsTheCorrectPassword() {
        String encoded = PasswordHasher.hash("correct horse battery".toCharArray());

        assertTrue(PasswordHasher.verify("correct horse battery".toCharArray(), encoded));
    }

    @Test
    public void verifyRejectsTheWrongPassword() {
        String encoded = PasswordHasher.hash("correct horse battery".toCharArray());

        assertFalse(PasswordHasher.verify("Correct horse battery".toCharArray(), encoded));
        assertFalse(PasswordHasher.verify("".toCharArray(), encoded));
    }

    @Test
    public void storedValueIsNotThePlaintext() {
        String password = "plaintextpassword";
        String encoded = PasswordHasher.hash(password.toCharArray());

        assertFalse(encoded.contains(password));
    }

    @Test
    public void samePasswordHashesDifferentlyEachTime() {
        // A per-password salt gives each one a different stored value, so a precomputed
        // table does not help an attacker.
        String first = PasswordHasher.hash("samepassword".toCharArray());
        String second = PasswordHasher.hash("samepassword".toCharArray());

        assertNotEquals(first, second);
        assertTrue(PasswordHasher.verify("samepassword".toCharArray(), first));
        assertTrue(PasswordHasher.verify("samepassword".toCharArray(), second));
    }

    @Test
    public void encodedValueCarriesItsFormatAndCost() {
        String[] fields = PasswordHasher.hash("somepassword".toCharArray()).split("\\$");

        assertEquals(4, fields.length);
        assertEquals("pbkdf2_sha256", fields[0]);
        assertTrue(Integer.parseInt(fields[1]) > 0);
    }

    @Test
    public void malformedStoredValuesNeverMatch() {
        assertFalse(PasswordHasher.verify("anything".toCharArray(), ""));
        assertFalse(PasswordHasher.verify("anything".toCharArray(), "plaintext"));
        assertFalse(PasswordHasher.verify("anything".toCharArray(), "pbkdf2_sha256$1$a"));
        assertFalse(PasswordHasher.verify("anything".toCharArray(), "other$1$YWJj$YWJj"));
        assertFalse(PasswordHasher.verify("anything".toCharArray(), "pbkdf2_sha256$0$YWJj$YWJj"));
        assertFalse(PasswordHasher.verify("anything".toCharArray(), null));
        assertFalse(PasswordHasher.verify(null, "pbkdf2_sha256$1$YWJj$YWJj"));
    }

    @Test
    public void unmatchableHashIsWellFormedAndMatchesNothing() {
        String encoded = PasswordHasher.unmatchableHash();

        assertEquals(4, encoded.split("\\$").length);
        assertFalse(PasswordHasher.verify("".toCharArray(), encoded));
        assertFalse(PasswordHasher.verify("password".toCharArray(), encoded));
    }

    @Test
    public void wipeClearsTheArray() {
        char[] password = "secretvalue".toCharArray();
        PasswordHasher.wipe(password);

        for (char c : password) {
            assertEquals('\0', c);
        }
    }

    @Test
    public void toCharArrayCopiesTheSequence() {
        assertEquals(0, PasswordHasher.toCharArray(null).length);
        assertEquals(0, PasswordHasher.toCharArray("").length);

        StringBuilder editable = new StringBuilder("hunter2xyz");
        char[] copied = PasswordHasher.toCharArray(editable);

        assertEquals("hunter2xyz", new String(copied));
    }
}
```

</details>

<details markdown="1">
<summary><code>UserRepositoryTest.java</code></summary>

```java
package com.example.cs360project.data;

import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertFalse;
import static org.junit.Assert.assertNotNull;
import static org.junit.Assert.assertNull;
import static org.junit.Assert.assertTrue;

import com.example.cs360project.data.dao.UserDao;
import com.example.cs360project.data.model.AuthenticatedUser;
import com.example.cs360project.data.model.User;
import com.example.cs360project.util.AppExecutors;
import com.example.cs360project.util.Outcome;

import org.junit.Before;
import org.junit.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.Executor;
import java.util.concurrent.atomic.AtomicReference;

/**
 * Tests for the authentication repository.
 *
 * The DAO is an interface and the executors arrive through a constructor, so a fake
 * store and a pair of same thread executors are enough to drive the real registration and
 * sign in logic on the JVM, with no emulator.
 */
public class UserRepositoryTest {

    /** In-memory {@link UserDao} standing in for SQLite. */
    private static class FakeUserDao implements UserDao {

        private final List<User> rows = new ArrayList<>();
        private long nextId = 1L;
        private boolean failInserts;

        @Override
        public long insert(String username, String encodedPassword) {
            if (failInserts) {
                return -1L;
            }
            User user = new User(nextId++, username, encodedPassword);
            rows.add(user);
            return user.getId();
        }

        @Override
        public User findByUsername(String username) {
            for (User user : rows) {
                if (user.getUsername().equals(username)) {
                    return user;
                }
            }
            return null;
        }

        String storedPasswordFor(String username) {
            User user = findByUsername(username);
            return user == null ? null : user.getEncodedPassword();
        }
    }

    private FakeUserDao dao;
    private UserRepository repository;

    @Before
    public void setUp() {
        dao = new FakeUserDao();
        Executor sameThread = Runnable::run;
        repository = new UserRepository(dao, new AppExecutors(sameThread, sameThread));
    }

    @Test
    public void registerStoresAHashRatherThanThePlaintext() {
        AtomicReference<Outcome<Void>> result = new AtomicReference<>();
        repository.register("david", "hunter2hunter2".toCharArray(), result::set);

        assertTrue(result.get().isSuccess());

        String stored = dao.storedPasswordFor("david");
        assertNotNull(stored);
        assertFalse(stored.contains("hunter2hunter2"));
        assertTrue(stored.startsWith("pbkdf2_sha256$"));
    }

    @Test
    public void registerClearsTheCallersPasswordArray() {
        char[] password = "hunter2hunter2".toCharArray();
        repository.register("david", password, outcome -> { });

        for (char c : password) {
            assertEquals('\0', c);
        }
    }

    @Test
    public void registerRejectsADuplicateUsername() {
        repository.register("david", "hunter2hunter2".toCharArray(), outcome -> { });

        AtomicReference<Outcome<Void>> result = new AtomicReference<>();
        repository.register("david", "differentpass".toCharArray(), result::set);

        assertFalse(result.get().isSuccess());
        assertNotNull(result.get().getMessage());
    }

    @Test
    public void registerReportsAFailedWrite() {
        dao.failInserts = true;

        AtomicReference<Outcome<Void>> result = new AtomicReference<>();
        repository.register("david", "hunter2hunter2".toCharArray(), result::set);

        // Discarding the insert return value makes a failed write look identical to a
        // successful one, so the outcome has to carry it.
        assertFalse(result.get().isSuccess());
    }

    @Test
    public void loginSucceedsWithTheCorrectPassword() {
        repository.register("david", "hunter2hunter2".toCharArray(), outcome -> { });

        AtomicReference<Outcome<AuthenticatedUser>> result = new AtomicReference<>();
        repository.login("david", "hunter2hunter2".toCharArray(), result::set);

        assertTrue(result.get().isSuccess());
        AuthenticatedUser user = result.get().getValue();
        assertEquals("david", user.getUsername());
        assertTrue(user.getId() > 0);
    }

    @Test
    public void loginFailsWithTheWrongPassword() {
        repository.register("david", "hunter2hunter2".toCharArray(), outcome -> { });

        AtomicReference<Outcome<AuthenticatedUser>> result = new AtomicReference<>();
        repository.login("david", "wrongpassword".toCharArray(), result::set);

        assertFalse(result.get().isSuccess());
        assertNull(result.get().getValue());
    }

    @Test
    public void unknownUsernameFailsWithTheSameMessageAsAWrongPassword() {
        repository.register("david", "hunter2hunter2".toCharArray(), outcome -> { });

        AtomicReference<Outcome<AuthenticatedUser>> wrongPassword = new AtomicReference<>();
        repository.login("david", "wrongpassword".toCharArray(), wrongPassword::set);

        AtomicReference<Outcome<AuthenticatedUser>> unknownUser = new AtomicReference<>();
        repository.login("nobody", "wrongpassword".toCharArray(), unknownUser::set);

        assertFalse(unknownUser.get().isSuccess());
        assertEquals(wrongPassword.get().getMessage(), unknownUser.get().getMessage());
    }

    @Test
    public void loginClearsTheCallersPasswordArray() {
        repository.register("david", "hunter2hunter2".toCharArray(), outcome -> { });

        char[] password = "hunter2hunter2".toCharArray();
        repository.login("david", password, outcome -> { });

        for (char c : password) {
            assertEquals('\0', c);
        }
    }
}
```

</details>

<details markdown="1">
<summary><code>WeightAnalyticsTest.java</code></summary>

```java
package com.example.cs360project.analytics;

import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertFalse;
import static org.junit.Assert.assertNotNull;
import static org.junit.Assert.assertNull;
import static org.junit.Assert.assertTrue;

import org.junit.Test;

/**
 * Correctness tests for the analytics calculations.
 *
 * These run on the JVM. Every method under test is a pure function with no Android
 * dependency.
 */
public class WeightAnalyticsTest {

    private static final float EPSILON = 0.0001f;

    // Moving average

    @Test
    public void movingAverageMatchesHandCalculatedValues() {
        float[] values = {10f, 20f, 30f, 40f};
        float[] result = WeightAnalytics.movingAverage(values, 2);

        // Positions before the window fills average over what exists.
        assertEquals(10f, result[0], EPSILON);
        assertEquals(15f, result[1], EPSILON);
        assertEquals(25f, result[2], EPSILON);
        assertEquals(35f, result[3], EPSILON);
    }

    @Test
    public void movingAverageAgreesWithNaiveRecomputation() {
        float[] values = new float[500];
        for (int i = 0; i < values.length; i++) {
            values[i] = (float) (200 + Math.sin(i / 7.0) * 12 - i * 0.05);
        }
        int window = 30;

        float[] fast = WeightAnalytics.movingAverage(values, window);

        // The O(n*k) version the running sum replaces. If the two disagree, the running
        // sum has drifted.
        for (int i = 0; i < values.length; i++) {
            int start = Math.max(0, i - window + 1);
            double sum = 0d;
            for (int j = start; j <= i; j++) {
                sum += values[j];
            }
            float naive = (float) (sum / (i - start + 1));
            assertEquals("index " + i, naive, fast[i], 0.01f);
        }
    }

    @Test
    public void movingAverageHandlesEdgeInputs() {
        assertEquals(0, WeightAnalytics.movingAverage(new float[0], 5).length);
        assertEquals(0, WeightAnalytics.movingAverage(null, 5).length);

        float[] single = WeightAnalytics.movingAverage(new float[]{42f}, 30);
        assertEquals(1, single.length);
        assertEquals(42f, single[0], EPSILON);
    }

    @Test
    public void movingAverageWindowOfOneReturnsTheInput() {
        float[] values = {1f, 2f, 3f};
        float[] result = WeightAnalytics.movingAverage(values, 1);
        assertEquals(1f, result[0], EPSILON);
        assertEquals(2f, result[1], EPSILON);
        assertEquals(3f, result[2], EPSILON);
    }

    @Test(expected = IllegalArgumentException.class)
    public void movingAverageRejectsWindowBelowOne() {
        WeightAnalytics.movingAverage(new float[]{1f}, 0);
    }

    // Streaks

    @Test
    public void detectsADownwardStreak() {
        StreakSummary s = WeightAnalytics.detectStreaks(new float[]{200f, 199f, 198f, 197f});

        assertEquals(StreakSummary.Direction.DOWN, s.getCurrentDirection());
        assertEquals(4, s.getCurrentLength());
        assertEquals(4, s.getLongestDownward());
        assertEquals(0, s.getLongestUpward());
    }

    @Test
    public void streakResetsWhenDirectionChanges() {
        StreakSummary s = WeightAnalytics.detectStreaks(new float[]{200f, 199f, 198f, 201f});

        assertEquals(StreakSummary.Direction.UP, s.getCurrentDirection());
        assertEquals(2, s.getCurrentLength());
        assertEquals(3, s.getLongestDownward());
    }

    @Test
    public void equalReadingBreaksTheStreak() {
        StreakSummary s = WeightAnalytics.detectStreaks(new float[]{200f, 199f, 199f});

        assertEquals(StreakSummary.Direction.FLAT, s.getCurrentDirection());
        assertEquals(2, s.getLongestDownward());
    }

    @Test
    public void streakHandlesEmptyAndSingleSeries() {
        assertEquals(StreakSummary.Direction.NONE,
                WeightAnalytics.detectStreaks(new float[0]).getCurrentDirection());
        assertEquals(StreakSummary.Direction.NONE,
                WeightAnalytics.detectStreaks(null).getCurrentDirection());

        StreakSummary single = WeightAnalytics.detectStreaks(new float[]{180f});
        assertEquals(1, single.getCurrentLength());
        assertEquals(0, single.getLongestDownward());
    }

    // Goal projection

    @Test
    public void projectsEntriesToGoalOnASteadyDecline() {
        // Losing exactly one pound per entry from 200, goal 195.
        float[] values = {200f, 199f, 198f, 197f, 196f};
        GoalProjection p = WeightAnalytics.projectToGoal(values, 5, 195f);

        assertTrue(p.isAvailable());
        assertEquals(-1f, p.getSlopePerEntry(), 0.001f);
        assertNotNull(p.getEntriesToGoal());
        assertEquals(1f, p.getEntriesToGoal(), 0.001f);
        assertEquals(195f, p.getProjectedNextValue(), 0.001f);
    }

    @Test
    public void reportsNoEntryCountWhenTrendMovesAwayFromGoal() {
        float[] values = {190f, 191f, 192f, 193f};
        GoalProjection p = WeightAnalytics.projectToGoal(values, 4, 180f);

        assertTrue(p.isAvailable());
        assertTrue(p.getSlopePerEntry() > 0);
        assertNull(p.getEntriesToGoal());
    }

    @Test
    public void reportsNoEntryCountWhenTrendIsFlat() {
        float[] values = {185f, 185f, 185f, 185f};
        GoalProjection p = WeightAnalytics.projectToGoal(values, 4, 180f);

        assertNull(p.getEntriesToGoal());
    }

    @Test
    public void projectionUsesOnlyTheRecentWindow() {
        // A sharp early climb followed by a steady decline. Fitting everything would show
        // a much weaker downward slope than fitting the recent entries.
        float[] values = {150f, 170f, 190f, 210f, 209f, 208f, 207f, 206f};

        GoalProjection recent = WeightAnalytics.projectToGoal(values, 4, 200f);
        GoalProjection whole = WeightAnalytics.projectToGoal(values, values.length, 200f);

        assertTrue(recent.getSlopePerEntry() < 0);
        assertTrue(whole.getSlopePerEntry() > 0);
    }

    @Test
    public void projectionNeedsEnoughData() {
        assertFalse(WeightAnalytics.projectToGoal(new float[]{200f}, 5, 190f).isAvailable());
        assertFalse(WeightAnalytics.projectToGoal(null, 5, 190f).isAvailable());
        assertFalse(WeightAnalytics.projectToGoal(new float[]{200f, 199f}, 1, 190f).isAvailable());
    }

    @Test
    public void goalAlreadyReachedReportsZero() {
        float[] values = {200f, 198f, 196f, 194f};
        GoalProjection p = WeightAnalytics.projectToGoal(values, 4, 199f);

        assertNotNull(p.getEntriesToGoal());
        assertEquals(0f, p.getEntriesToGoal(), EPSILON);
    }
}
```

</details>

<details markdown="1">
<summary><code>MonotonicWindowTest.java</code></summary>

```java
package com.example.cs360project.analytics;

import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertTrue;

import org.junit.Test;

import java.util.Random;

/**
 * Tests for the sliding window minimum and maximum.
 *
 * Answers are checked against a brute force scan. Cost is checked by counting deque
 * operations, which gives the same result on any machine.
 */
public class MonotonicWindowTest {

    @Test
    public void matchesBruteForceOnRandomData() {
        Random random = new Random(20260927L);
        int n = 2000;
        int windowSize = 30;

        float[] values = new float[n];
        for (int i = 0; i < n; i++) {
            values[i] = 150f + random.nextInt(1000) / 10f;
        }

        MonotonicWindow window = new MonotonicWindow(windowSize);
        for (int i = 0; i < n; i++) {
            window.append(values[i]);

            int start = Math.max(0, i - windowSize + 1);
            float expectedMin = Float.MAX_VALUE;
            float expectedMax = -Float.MAX_VALUE;
            for (int j = start; j <= i; j++) {
                expectedMin = Math.min(expectedMin, values[j]);
                expectedMax = Math.max(expectedMax, values[j]);
            }

            assertEquals("min at " + i, expectedMin, window.currentMin(), 0.0001f);
            assertEquals("max at " + i, expectedMax, window.currentMax(), 0.0001f);
        }
    }

    @Test
    public void handlesStrictlyDecreasingInput() {
        // Worst case for the maximum deque: nothing is ever discarded on append.
        MonotonicWindow window = new MonotonicWindow(3);
        for (int i = 10; i > 0; i--) {
            window.append(i);
        }
        assertEquals(1f, window.currentMin(), 0.0001f);
        assertEquals(3f, window.currentMax(), 0.0001f);
    }

    @Test
    public void handlesStrictlyIncreasingInput() {
        MonotonicWindow window = new MonotonicWindow(3);
        for (int i = 1; i <= 10; i++) {
            window.append(i);
        }
        assertEquals(8f, window.currentMin(), 0.0001f);
        assertEquals(10f, window.currentMax(), 0.0001f);
    }

    @Test
    public void handlesRepeatedValues() {
        MonotonicWindow window = new MonotonicWindow(4);
        for (int i = 0; i < 6; i++) {
            window.append(180f);
        }
        assertEquals(180f, window.currentMin(), 0.0001f);
        assertEquals(180f, window.currentMax(), 0.0001f);
    }

    @Test
    public void windowOfOneTracksTheLatestValue() {
        MonotonicWindow window = new MonotonicWindow(1);
        window.append(200f);
        window.append(150f);
        assertEquals(150f, window.currentMin(), 0.0001f);
        assertEquals(150f, window.currentMax(), 0.0001f);
    }

    @Test(expected = IllegalArgumentException.class)
    public void rejectsWindowBelowOne() {
        new MonotonicWindow(0);
    }

    @Test(expected = IllegalStateException.class)
    public void readingBeforeAnyAppendFails() {
        new MonotonicWindow(5).currentMin();
    }

    /**
     * The complexity claim, asserted by counting rather than timing.
     *
     * Each index is pushed once and popped at most once per deque, so two deques plus
     * evictions give a ceiling of six operations per element. A rescan would do work
     * proportional to the window, failing this bound for any window wider than six.
     */
    @Test
    public void totalOperationsStayLinearRegardlessOfWindowSize() {
        Random random = new Random(7L);
        int n = 50_000;

        float[] values = new float[n];
        for (int i = 0; i < n; i++) {
            values[i] = random.nextFloat() * 100f;
        }

        for (int windowSize : new int[]{2, 30, 365, 5000}) {
            MonotonicWindow window = new MonotonicWindow(windowSize);
            for (float value : values) {
                window.append(value);
            }

            long operations = window.operationCount();
            assertTrue("window " + windowSize + " used " + operations
                            + " operations for " + n + " elements",
                    operations <= 6L * n);
        }
    }

    /**
     * The same bound stated as a ratio, so widening the window does not change the cost.
     *
     * This is the assertion that actually separates the deque from a rescan. If the
     * implementation were replaced with a scan, a window of 5000 would cost roughly 2500
     * times what a window of 2 costs. Here the two are within a small constant factor.
     */
    @Test
    public void costDoesNotGrowWithWindowSize() {
        Random random = new Random(11L);
        int n = 20_000;
        float[] values = new float[n];
        for (int i = 0; i < n; i++) {
            values[i] = random.nextFloat() * 100f;
        }

        MonotonicWindow narrow = new MonotonicWindow(2);
        MonotonicWindow wide = new MonotonicWindow(5000);
        for (float value : values) {
            narrow.append(value);
            wide.append(value);
        }

        double ratio = (double) wide.operationCount() / narrow.operationCount();
        assertTrue("wide/narrow operation ratio was " + ratio, ratio < 2.0d);
    }
}
```

</details>

<details markdown="1">
<summary><code>AnalyticsEngineTest.java</code></summary>

```java
package com.example.cs360project.analytics;

import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertFalse;
import static org.junit.Assert.assertNotNull;
import static org.junit.Assert.assertTrue;

import org.junit.Test;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Random;

/**
 * Tests for the engine that keeps the analytics current.
 *
 * The important one is {@link #incrementalAppendAgreesWithFullRebuild()}. The engine has
 * two update paths, a fast incremental append and a full rebuild used on edit and delete,
 * and the risk with two paths is that they quietly disagree. Replaying the same series both
 * ways and comparing every figure is what rules that out.
 */
public class AnalyticsEngineTest {

    private static final float EPSILON = 0.001f;

    private static List<Float> series(float... values) {
        List<Float> list = new ArrayList<>();
        for (float value : values) {
            list.add(value);
        }
        return list;
    }

    @Test
    public void emptyEngineReportsNoData() {
        AnalyticsSnapshot snapshot = new AnalyticsEngine().snapshot();

        assertFalse(snapshot.hasData());
        assertEquals(0, snapshot.getEntryCount());
        assertFalse(snapshot.getProjection().isAvailable());
    }

    @Test
    public void singleEntryReportsItselfEverywhere() {
        AnalyticsEngine engine = new AnalyticsEngine();
        engine.append(185f);

        AnalyticsSnapshot snapshot = engine.snapshot();
        assertTrue(snapshot.hasData());
        assertEquals(1, snapshot.getEntryCount());
        assertEquals(185f, snapshot.getLatestValue(), EPSILON);
        assertEquals(185f, snapshot.getMovingAverage(), EPSILON);
        assertEquals(185f, snapshot.getWindowMin(), EPSILON);
        assertEquals(185f, snapshot.getWindowMax(), EPSILON);
    }

    @Test
    public void incrementalAppendAgreesWithFullRebuild() {
        Random random = new Random(20260927L);
        List<Float> values = new ArrayList<>();
        for (int i = 0; i < 1500; i++) {
            values.add(180f + random.nextInt(400) / 10f);
        }

        AnalyticsEngine appended = new AnalyticsEngine();
        appended.setGoal(170f);
        for (Float value : values) {
            appended.append(value);
        }

        AnalyticsEngine rebuilt = new AnalyticsEngine();
        rebuilt.setGoal(170f);
        rebuilt.rebuild(values);

        assertSnapshotsMatch(appended.snapshot(), rebuilt.snapshot());
    }

    @Test
    public void rebuildAfterDeletionMatchesAFreshEngine() {
        List<Float> original = series(200f, 198f, 205f, 197f, 196f, 210f, 195f);

        AnalyticsEngine engine = new AnalyticsEngine();
        engine.setGoal(190f);
        engine.rebuild(original);

        // Remove the spike, the case the append path cannot handle.
        List<Float> afterDelete = new ArrayList<>(original);
        afterDelete.remove(Float.valueOf(210f));
        engine.rebuild(afterDelete);

        AnalyticsEngine fresh = new AnalyticsEngine();
        fresh.setGoal(190f);
        fresh.rebuild(afterDelete);

        assertSnapshotsMatch(engine.snapshot(), fresh.snapshot());
    }

    @Test
    public void rebuildAfterEditMatchesAFreshEngine() {
        AnalyticsEngine engine = new AnalyticsEngine();
        engine.rebuild(series(200f, 199f, 198f));

        List<Float> edited = series(200f, 250f, 198f);
        engine.rebuild(edited);

        AnalyticsEngine fresh = new AnalyticsEngine();
        fresh.rebuild(edited);

        assertSnapshotsMatch(engine.snapshot(), fresh.snapshot());
        assertEquals(250f, engine.snapshot().getWindowMax(), EPSILON);
    }

    @Test
    public void rebuildClearsPreviousState() {
        AnalyticsEngine engine = new AnalyticsEngine();
        engine.rebuild(series(300f, 299f, 298f));
        engine.rebuild(series(150f, 151f));

        AnalyticsSnapshot snapshot = engine.snapshot();
        assertEquals(2, snapshot.getEntryCount());
        assertEquals(150f, snapshot.getWindowMin(), EPSILON);
        assertEquals(151f, snapshot.getWindowMax(), EPSILON);
        assertEquals(StreakSummary.Direction.UP, snapshot.getStreaks().getCurrentDirection());
    }

    @Test
    public void rebuildToleratesNullAndEmptySeries() {
        AnalyticsEngine engine = new AnalyticsEngine();
        engine.rebuild(series(200f, 199f));

        engine.rebuild(null);
        assertFalse(engine.snapshot().hasData());

        engine.rebuild(new ArrayList<>());
        assertFalse(engine.snapshot().hasData());
    }

    @Test
    public void movingAverageIgnoresEntriesOutsideTheWindow() {
        AnalyticsEngine engine = new AnalyticsEngine(3, 3);
        engine.rebuild(series(100f, 100f, 100f, 200f, 200f, 200f));

        // The first three entries have left the window entirely.
        assertEquals(200f, engine.snapshot().getMovingAverage(), EPSILON);
    }

    @Test
    public void goalProjectionAppearsOnceAGoalIsSet() {
        AnalyticsEngine engine = new AnalyticsEngine();
        engine.rebuild(series(200f, 199f, 198f, 197f));

        assertFalse(engine.snapshot().getProjection().isAvailable());

        engine.setGoal(195f);
        GoalProjection projection = engine.snapshot().getProjection();
        assertTrue(projection.isAvailable());
        assertNotNull(projection.getEntriesToGoal());
    }

    @Test
    public void appendWorkDoesNotGrowWithHistoryLength() {
        // The claim behind the incremental path: logging the ten thousandth entry costs
        // the same as logging the tenth.
        AnalyticsEngine engine = new AnalyticsEngine();
        Random random = new Random(3L);

        for (int i = 0; i < 1000; i++) {
            engine.append(random.nextFloat() * 50f + 150f);
        }
        long afterFirstThousand = engine.windowOperationCount();

        for (int i = 0; i < 1000; i++) {
            engine.append(random.nextFloat() * 50f + 150f);
        }
        long afterSecondThousand = engine.windowOperationCount();

        long firstBatch = afterFirstThousand;
        long secondBatch = afterSecondThousand - afterFirstThousand;

        // Both batches are 1000 appends. If cost grew with history, the second would be
        // dramatically larger.
        double ratio = (double) secondBatch / firstBatch;
        assertTrue("second batch cost ratio was " + ratio, ratio > 0.5d && ratio < 2.0d);
    }

    @Test
    public void movingAverageSeriesMatchesTheStandaloneFunction() {
        List<Float> values = series(190f, 191f, 189f, 188f, 192f, 187f);
        AnalyticsEngine engine = new AnalyticsEngine(3, 3);
        engine.rebuild(values);

        float[] expected = WeightAnalytics.movingAverage(
                new float[]{190f, 191f, 189f, 188f, 192f, 187f}, 3);

        assertTrue(Arrays.equals(expected, engine.movingAverageSeries()));
    }

    private static void assertSnapshotsMatch(AnalyticsSnapshot a, AnalyticsSnapshot b) {
        assertEquals(a.getEntryCount(), b.getEntryCount());
        assertEquals(a.getLatestValue(), b.getLatestValue(), EPSILON);
        assertEquals(a.getMovingAverage(), b.getMovingAverage(), EPSILON);
        assertEquals(a.getWindowMin(), b.getWindowMin(), EPSILON);
        assertEquals(a.getWindowMax(), b.getWindowMax(), EPSILON);
        assertEquals(a.getStreaks().getCurrentDirection(), b.getStreaks().getCurrentDirection());
        assertEquals(a.getStreaks().getCurrentLength(), b.getStreaks().getCurrentLength());
        assertEquals(a.getStreaks().getLongestDownward(), b.getStreaks().getLongestDownward());
        assertEquals(a.getStreaks().getLongestUpward(), b.getStreaks().getLongestUpward());
        assertEquals(a.getProjection().isAvailable(), b.getProjection().isAvailable());
        assertEquals(a.getProjection().getSlopePerEntry(),
                b.getProjection().getSlopePerEntry(), EPSILON);
    }
}
```

</details>

<details markdown="1">
<summary><code>LegacyTimestampParserTest.java</code></summary>

```java
package com.example.cs360project.data.migration;

import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertNotNull;
import static org.junit.Assert.assertNull;
import static org.junit.Assert.assertTrue;

import org.junit.Test;

import java.text.SimpleDateFormat;
import java.util.Locale;
import java.util.TimeZone;

/**
 * Tests for reading the timestamps the first version wrote.
 *
 * This is the part of the migration most likely to be wrong, because the input is display
 * text that was never meant to be read back. Covering it here means the behavior is known
 * before an upgrade runs on somebody's phone, where a bad conversion is both silent and
 * permanent.
 */
public class LegacyTimestampParserTest {

    private static long epochOf(String iso) throws Exception {
        SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss", Locale.US);
        format.setTimeZone(TimeZone.getTimeZone("UTC"));
        return format.parse(iso).getTime();
    }

    @Test
    public void readsAnEnglishTimestamp() throws Exception {
        Long parsed = LegacyTimestampParser.parse("Mon Sep 07 14:23:11 UTC 2026");

        assertNotNull(parsed);
        assertEquals(epochOf("2026-09-07 14:23:11"), (long) parsed);
    }

    @Test
    public void honoursTheZoneInTheText() throws Exception {
        Long utc = LegacyTimestampParser.parse("Mon Sep 07 14:23:11 UTC 2026");
        Long gmt = LegacyTimestampParser.parse("Mon Sep 07 14:23:11 GMT 2026");

        assertNotNull(utc);
        assertNotNull(gmt);
        assertEquals((long) utc, (long) gmt);
        assertEquals(epochOf("2026-09-07 14:23:11"), (long) utc);
    }

    @Test
    public void returnsNullForTextItCannotRead() {
        assertNull(LegacyTimestampParser.parse(null));
        assertNull(LegacyTimestampParser.parse(""));
        assertNull(LegacyTimestampParser.parse("   "));
        assertNull(LegacyTimestampParser.parse("not a date"));
        assertNull(LegacyTimestampParser.parse("2026-09-07"));
    }

    @Test
    public void fallbackIsUsedOnlyWhenParsingFails() {
        long fallback = 42L;

        assertEquals(fallback,
                LegacyTimestampParser.parseOrFallback("unreadable", fallback));
        assertTrue(LegacyTimestampParser.parseOrFallback(
                "Mon Sep 07 14:23:11 UTC 2026", fallback) != fallback);
    }

    @Test
    public void sequenceKeepsReadableValuesUntouched() {
        Long[] parsed = {1000L, 2000L, 3000L};
        long[] resolved = LegacyTimestampParser.resolveSequence(parsed, 0L);

        assertEquals(1000L, resolved[0]);
        assertEquals(2000L, resolved[1]);
        assertEquals(3000L, resolved[2]);
    }

    @Test
    public void sequenceFillsAGapFromTheRowBefore() {
        Long[] parsed = {1000L, null, 3000L};
        long[] resolved = LegacyTimestampParser.resolveSequence(parsed, 0L);

        assertEquals(1000L, resolved[0]);
        assertEquals(1001L, resolved[1]);
        assertEquals(3000L, resolved[2]);
        assertTrue(resolved[0] < resolved[1] && resolved[1] < resolved[2]);
    }

    @Test
    public void sequenceFillsALeadingGapBelowTheFirstKnownValue() {
        Long[] parsed = {null, null, 3000L};
        long[] resolved = LegacyTimestampParser.resolveSequence(parsed, 0L);

        // Unreadable rows at the front stay at the front rather than jumping ahead.
        assertEquals(2998L, resolved[0]);
        assertEquals(2999L, resolved[1]);
        assertEquals(3000L, resolved[2]);
    }

    @Test
    public void sequenceHandlesEverythingUnreadable() {
        Long[] parsed = {null, null, null};
        long[] resolved = LegacyTimestampParser.resolveSequence(parsed, 5000L);

        assertEquals(5000L, resolved[0]);
        assertEquals(5001L, resolved[1]);
        assertEquals(5002L, resolved[2]);
    }

    @Test
    public void sequenceIsAlwaysStrictlyIncreasing() {
        // Two entries logged in the same second, which the old format could not separate.
        Long[] parsed = {1000L, 1000L, 900L, null, 1000L};
        long[] resolved = LegacyTimestampParser.resolveSequence(parsed, 0L);

        for (int i = 1; i < resolved.length; i++) {
            assertTrue("index " + i + " of " + java.util.Arrays.toString(resolved),
                    resolved[i] > resolved[i - 1]);
        }
    }

    @Test
    public void sequenceHandlesEmptyInput() {
        assertEquals(0, LegacyTimestampParser.resolveSequence(new Long[0], 0L).length);
        assertEquals(0, LegacyTimestampParser.resolveSequence(null, 0L).length);
    }

    @Test
    public void patternIsTheOneDateToStringProduces() {
        // Guards against the pattern drifting away from what the old code actually wrote.
        assertEquals("EEE MMM dd HH:mm:ss zzz yyyy", LegacyTimestampParser.legacyPattern());
    }
}
```

</details>

<details markdown="1">
<summary><code>LegacyCredentialTest.java</code></summary>

```java
package com.example.cs360project.data.migration;

import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertFalse;
import static org.junit.Assert.assertNull;
import static org.junit.Assert.assertTrue;

import org.junit.Test;

/**
 * Tests for splitting a packed credential into schema columns.
 *
 * The case worth protecting is the row that is not in the packed format at all. An install
 * old enough to predate hashing has a plaintext password sitting in that column, and
 * copying it into one named passwordHash would turn a visible problem into an invisible
 * one.
 */
public class LegacyCredentialTest {

    private static final String PACKED = "pbkdf2_sha256$120000$c2FsdA==$aGFzaA==";

    @Test
    public void splitsAPackedCredential() {
        LegacyCredential credential = LegacyCredential.split(PACKED);

        assertTrue(credential.isParsed());
        assertEquals("pbkdf2_sha256", credential.getAlgorithm());
        assertEquals(120000, credential.getIterations());
        assertEquals("c2FsdA==", credential.getSalt());
        assertEquals("aGFzaA==", credential.getHash());
    }

    @Test
    public void roundTripsBackToThePackedForm() {
        assertEquals(PACKED, LegacyCredential.split(PACKED).toPacked());
    }

    @Test
    public void plaintextIsFlaggedRatherThanTreatedAsAHash() {
        LegacyCredential credential = LegacyCredential.split("hunter2");

        assertFalse(credential.isParsed());
        assertEquals("unmigrated", credential.getAlgorithm());
        assertEquals(0, credential.getIterations());
        // The stored text is kept so nothing is thrown away.
        assertEquals("hunter2", credential.getHash());
        // No packed form means no password can verify against it.
        assertNull(credential.toPacked());
    }

    @Test
    public void rejectsMalformedPackedValues() {
        assertFalse(LegacyCredential.split(null).isParsed());
        assertFalse(LegacyCredential.split("").isParsed());
        assertFalse(LegacyCredential.split("a$b$c").isParsed());
        assertFalse(LegacyCredential.split("a$b$c$d$e").isParsed());
        assertFalse(LegacyCredential.split("alg$notanumber$salt$hash").isParsed());
        assertFalse(LegacyCredential.split("alg$0$salt$hash").isParsed());
        assertFalse(LegacyCredential.split("alg$-5$salt$hash").isParsed());
        assertFalse(LegacyCredential.split("$120000$salt$hash").isParsed());
        assertFalse(LegacyCredential.split("alg$120000$$hash").isParsed());
        assertFalse(LegacyCredential.split("alg$120000$salt$").isParsed());
    }

    @Test
    public void buildsFromColumnsAlreadySplit() {
        LegacyCredential credential =
                LegacyCredential.of("pbkdf2_sha256", 120000, "c2FsdA==", "aGFzaA==");

        assertTrue(credential.isParsed());
        assertEquals(PACKED, credential.toPacked());
    }

    @Test
    public void columnsThatCameFromAnUnmigratedRowStayUnusable() {
        LegacyCredential credential =
                LegacyCredential.of("unmigrated", 0, "", "hunter2");

        assertFalse(credential.isParsed());
        assertNull(credential.toPacked());
    }

    @Test
    public void rejectsIncompleteColumns() {
        assertFalse(LegacyCredential.of(null, 120000, "s", "h").isParsed());
        assertFalse(LegacyCredential.of("alg", 0, "s", "h").isParsed());
        assertFalse(LegacyCredential.of("alg", 120000, "", "h").isParsed());
        assertFalse(LegacyCredential.of("alg", 120000, "s", null).isParsed());
    }
}
```

</details>
