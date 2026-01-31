# React Native Forms

Form handling patterns with react-hook-form and Zod validation.

## When to Apply

- Building registration/login forms
- Complex multi-step forms
- Form validation
- Dynamic form fields

## Installation

```bash
npm install react-hook-form zod @hookform/resolvers
```

## Basic Form

```tsx
import { useForm, Controller } from 'react-hook-form';
import { View, Text, TextInput, Button, StyleSheet } from 'react-native';

interface FormData {
  email: string;
  password: string;
}

function LoginForm() {
  const { control, handleSubmit, formState: { errors } } = useForm<FormData>({
    defaultValues: {
      email: '',
      password: '',
    },
  });

  const onSubmit = (data: FormData) => {
    console.log(data);
  };

  return (
    <View style={styles.form}>
      <Controller
        control={control}
        name="email"
        rules={{ required: 'Email is required' }}
        render={({ field: { onChange, onBlur, value } }) => (
          <TextInput
            style={styles.input}
            placeholder="Email"
            onBlur={onBlur}
            onChangeText={onChange}
            value={value}
            autoCapitalize="none"
            keyboardType="email-address"
          />
        )}
      />
      {errors.email && <Text style={styles.error}>{errors.email.message}</Text>}

      <Controller
        control={control}
        name="password"
        rules={{ required: 'Password is required' }}
        render={({ field: { onChange, onBlur, value } }) => (
          <TextInput
            style={styles.input}
            placeholder="Password"
            onBlur={onBlur}
            onChangeText={onChange}
            value={value}
            secureTextEntry
          />
        )}
      />
      {errors.password && <Text style={styles.error}>{errors.password.message}</Text>}

      <Button title="Login" onPress={handleSubmit(onSubmit)} />
    </View>
  );
}
```

## Zod Validation

```tsx
import { useForm, Controller } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const signupSchema = z.object({
  name: z.string().min(2, 'Name must be at least 2 characters'),
  email: z.string().email('Invalid email address'),
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .regex(/[A-Z]/, 'Password must contain an uppercase letter')
    .regex(/[0-9]/, 'Password must contain a number'),
  confirmPassword: z.string(),
}).refine((data) => data.password === data.confirmPassword, {
  message: "Passwords don't match",
  path: ['confirmPassword'],
});

type SignupFormData = z.infer<typeof signupSchema>;

function SignupForm() {
  const { control, handleSubmit, formState: { errors } } = useForm<SignupFormData>({
    resolver: zodResolver(signupSchema),
    defaultValues: {
      name: '',
      email: '',
      password: '',
      confirmPassword: '',
    },
  });

  const onSubmit = (data: SignupFormData) => {
    console.log(data);
  };

  return (
    <View style={styles.form}>
      {/* Form fields with Controller */}
    </View>
  );
}
```

## Reusable Input Component

```tsx
import { useController, UseControllerProps } from 'react-hook-form';
import { TextInput, Text, View, TextInputProps, StyleSheet } from 'react-native';

interface FormInputProps extends TextInputProps, UseControllerProps {
  label?: string;
}

function FormInput({ name, control, rules, label, ...inputProps }: FormInputProps) {
  const {
    field: { value, onChange, onBlur },
    fieldState: { error },
  } = useController({
    name,
    control,
    rules,
  });

  return (
    <View style={styles.inputContainer}>
      {label && <Text style={styles.label}>{label}</Text>}
      <TextInput
        style={[styles.input, error && styles.inputError]}
        value={value}
        onChangeText={onChange}
        onBlur={onBlur}
        {...inputProps}
      />
      {error && <Text style={styles.errorText}>{error.message}</Text>}
    </View>
  );
}

// Usage
<FormInput
  name="email"
  control={control}
  label="Email"
  placeholder="Enter email"
  keyboardType="email-address"
/>
```

## Form with Select/Picker

```tsx
import { Picker } from '@react-native-picker/picker';

function FormPicker({ name, control, items, label }) {
  const { field: { value, onChange }, fieldState: { error } } = useController({
    name,
    control,
  });

  return (
    <View style={styles.inputContainer}>
      {label && <Text style={styles.label}>{label}</Text>}
      <Picker
        selectedValue={value}
        onValueChange={onChange}
        style={styles.picker}
      >
        {items.map((item) => (
          <Picker.Item key={item.value} label={item.label} value={item.value} />
        ))}
      </Picker>
      {error && <Text style={styles.errorText}>{error.message}</Text>}
    </View>
  );
}
```

## Switch/Checkbox

```tsx
import { Switch } from 'react-native';

function FormSwitch({ name, control, label }) {
  const { field: { value, onChange } } = useController({
    name,
    control,
  });

  return (
    <View style={styles.switchContainer}>
      <Text style={styles.label}>{label}</Text>
      <Switch
        value={value}
        onValueChange={onChange}
        trackColor={{ false: '#767577', true: '#81b0ff' }}
        thumbColor={value ? '#f5dd4b' : '#f4f3f4'}
      />
    </View>
  );
}
```

## Multi-Step Form

```tsx
import { useState } from 'react';
import { useForm, FormProvider, useFormContext } from 'react-hook-form';

const steps = ['Personal', 'Contact', 'Review'];

function MultiStepForm() {
  const [currentStep, setCurrentStep] = useState(0);
  const methods = useForm({
    defaultValues: {
      firstName: '',
      lastName: '',
      email: '',
      phone: '',
    },
  });

  const onSubmit = (data) => {
    console.log('Final data:', data);
  };

  const nextStep = async () => {
    const isValid = await methods.trigger(getFieldsForStep(currentStep));
    if (isValid) {
      setCurrentStep((prev) => Math.min(prev + 1, steps.length - 1));
    }
  };

  const prevStep = () => {
    setCurrentStep((prev) => Math.max(prev - 1, 0));
  };

  return (
    <FormProvider {...methods}>
      <View>
        <StepIndicator steps={steps} currentStep={currentStep} />
        
        {currentStep === 0 && <PersonalInfoStep />}
        {currentStep === 1 && <ContactInfoStep />}
        {currentStep === 2 && <ReviewStep />}
        
        <View style={styles.buttons}>
          {currentStep > 0 && (
            <Button title="Back" onPress={prevStep} />
          )}
          {currentStep < steps.length - 1 ? (
            <Button title="Next" onPress={nextStep} />
          ) : (
            <Button title="Submit" onPress={methods.handleSubmit(onSubmit)} />
          )}
        </View>
      </View>
    </FormProvider>
  );
}

// Step component
function PersonalInfoStep() {
  const { control } = useFormContext();
  
  return (
    <View>
      <FormInput name="firstName" control={control} label="First Name" />
      <FormInput name="lastName" control={control} label="Last Name" />
    </View>
  );
}
```

## Dynamic Fields (Array)

```tsx
import { useFieldArray } from 'react-hook-form';

function DynamicForm() {
  const { control, handleSubmit } = useForm({
    defaultValues: {
      items: [{ name: '', quantity: 1 }],
    },
  });

  const { fields, append, remove } = useFieldArray({
    control,
    name: 'items',
  });

  return (
    <View>
      {fields.map((field, index) => (
        <View key={field.id} style={styles.row}>
          <FormInput
            name={`items.${index}.name`}
            control={control}
            placeholder="Item name"
          />
          <FormInput
            name={`items.${index}.quantity`}
            control={control}
            keyboardType="numeric"
            placeholder="Qty"
          />
          <Button title="Remove" onPress={() => remove(index)} />
        </View>
      ))}
      
      <Button 
        title="Add Item" 
        onPress={() => append({ name: '', quantity: 1 })} 
      />
    </View>
  );
}
```

## Async Validation

```tsx
const schema = z.object({
  username: z
    .string()
    .min(3)
    .refine(async (username) => {
      const available = await checkUsernameAvailability(username);
      return available;
    }, 'Username is already taken'),
});

// Or with rules
<Controller
  name="username"
  control={control}
  rules={{
    validate: async (value) => {
      const available = await checkUsernameAvailability(value);
      return available || 'Username is already taken';
    },
  }}
  render={({ field }) => <TextInput {...field} />}
/>
```

## Form State Handling

```tsx
function FormWithState() {
  const {
    control,
    handleSubmit,
    formState: {
      errors,
      isSubmitting,
      isValid,
      isDirty,
      touchedFields,
    },
    reset,
    watch,
    setValue,
  } = useForm();

  // Watch specific field
  const email = watch('email');

  // Watch all fields
  const allValues = watch();

  return (
    <View>
      {/* Form inputs */}
      
      <Button
        title={isSubmitting ? 'Submitting...' : 'Submit'}
        onPress={handleSubmit(onSubmit)}
        disabled={!isValid || isSubmitting}
      />
      
      <Button
        title="Reset"
        onPress={() => reset()}
        disabled={!isDirty}
      />
    </View>
  );
}
```

## Keyboard Handling

```tsx
import { KeyboardAvoidingView, Platform, ScrollView } from 'react-native';

function FormWithKeyboard() {
  return (
    <KeyboardAvoidingView
      behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
      style={styles.container}
    >
      <ScrollView
        contentContainerStyle={styles.scrollContent}
        keyboardShouldPersistTaps="handled"
      >
        {/* Form content */}
      </ScrollView>
    </KeyboardAvoidingView>
  );
}
```

## Best Practices

1. **Use Zod for validation** - Type-safe schemas
2. **Create reusable components** - FormInput, FormSelect, etc.
3. **Handle keyboard** - KeyboardAvoidingView on iOS
4. **Show loading states** - Disable button while submitting
5. **Validate on blur** - Mode 'onBlur' for better UX
6. **Use FormProvider** - For deeply nested forms
7. **Reset form properly** - Clear on successful submit
